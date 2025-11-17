# Crypto Watchdog Bot – Detailed Script Explanation

This document explains in detail what the `bot.js` script does for your **Crypto Watchdog** Discord bot.  
It is based on the version using the **CoinGecko Demo API**, with:

- intraday alerts (fast moves),
- daily trend alerts (vs daily open),
- new ATH alerts,
- and a daily summary recap.

---

## 1. Initialization: imports & Discord client

At the top of the file, the script imports:

- `dotenv/config`  
  → Automatically loads environment variables from the `.env` file (`DISCORD_TOKEN`, `COINGECKO_API_KEY`, etc.).

- `{ Client, GatewayIntentBits, EmbedBuilder }` from `discord.js`  
  → Used to connect to Discord, access guilds/channels, and build rich embed messages.

- `{ WATCHED_COINS }` from `./config/coins.js`  
  → The list of coins you want to monitor (BTC, ETH, SOL, XRP, DOGE, GMT) with their thresholds.

Then it creates a Discord client:

```js
const client = new Client({
  intents: [GatewayIntentBits.Guilds]
});
```

The bot only needs the **Guilds** intent, because it sends messages into a known channel by ID; it doesn’t need to read member messages.

---

## 2. Reading configuration from `.env`

The script reads several values from your `.env` file:

- `COINGECKO_API_KEY` → your CoinGecko **Demo API** key.  
- `DISCORD_CHANNEL_ID` → the ID of the Discord text channel where the bot will post alerts and summaries.  
- `CHECK_INTERVAL_MINUTES` → how often the bot checks prices (e.g., every 5 minutes).  
- `DAILY_SUMMARY_HOUR` and `DAILY_SUMMARY_MINUTE` → the time of day when the daily recap is posted.

It also defines:

```js
const BASE_URL = 'https://api.coingecko.com/api/v3';
```

This is the base URL for the **CoinGecko Demo API**.

---

## 3. In‑memory state while the bot is running

The script uses several `Map()` objects to keep state in memory between checks:

- `lastPrices`  
  → For each coin, stores the **last observed price**.  
  → Used to compute the percentage change over the last interval (intraday movement).

- `intradayCooldown`  
  → For each coin, stores the **timestamp** (in ms) of the last intraday alert sent.  
  → A cooldown (`intradayCooldownMs`) is set to 15 minutes to avoid spam.

- `dayOpenPrices`  
  → For each coin, stores `{ price, dayKey }` where:
    - `price` is the **opening price of the day**,
    - `dayKey` is the current day in UTC (e.g., `2025-11-17`).  
  → Used to compute the **daily change** vs the opening price.

- `dailyAlertState`  
  → For each coin, stores an object `{ up: bool, down: bool }`:
    - `up` = whether the **upward daily threshold** alert has already been sent today,
    - `down` = whether the **downward daily threshold** alert has already been sent today.  
  → This prevents daily trend alerts from being sent multiple times in one day.

- `athState`  
  → For each coin, stores:  
    `{ athUsd, athDate, athBuffer, lastAnnouncedAthPrice }`
    - `athUsd` → the current official ATH (All‑Time High) in USD from CoinGecko,
    - `athDate` → the date of this ATH (from CoinGecko),
    - `athBuffer` → a small margin (e.g., 0.05–0.1%) above `athUsd` used to define a “new ATH”,
    - `lastAnnouncedAthPrice` → the last price level for which a “new ATH” alert was already sent, to avoid spamming for each tiny tick above.

- `lastSummaryDayKey`  
  → Stores the day (UTC) for which today’s daily summary has already been sent, to avoid sending the recap multiple times.

---

## 4. Utility functions: current date & generic JSON fetch

### `getTodayKeyUtc()`

Returns today’s date in UTC as a string: `YYYY-MM-DD`.  
The script uses this to detect day changes and reset daily state.

### `fetchJson(pathAndQuery)`

Takes a path + query string (e.g. `"/simple/price?...")`, builds a full URL:

```js
const url = `${BASE_URL}${pathAndQuery}`;
```

Then performs a `fetch` with the required CoinGecko Demo header:

```js
headers: {
  'x-cg-demo-api-key': API_KEY
}
```

Behavior:

- If the HTTP response is not OK (`!res.ok`):
  - logs the status code, the URL, and the first part of the error body,
  - returns `null`.
- Otherwise, it returns `res.json()`.

All CoinGecko API requests go through this helper.

---

## 5. `fetchSimplePrices()` – live prices for all coins

This function:

1. Builds a comma‑separated list of CoinGecko IDs from `WATCHED_COINS`  
   (e.g., `bitcoin,ethereum,solana,ripple,dogecoin,gmt-token`).
2. Calls:

   ```txt
   /simple/price?ids=...&vs_currencies=usd&include_24hr_change=true
   ```

3. Returns an object of the form:

   ```js
   {
     bitcoin:  { usd: 65000, usd_24h_change: 2.3 },
     ethereum: { usd: 3500,  usd_24h_change: 3.1 },
     // ...
   }
   ```

This is used by the main monitoring loop to compute intraday and daily movements.

---

## 6. `refreshAthForAllCoins()` – updating ATH data (once per day)

This function is called:

- once when the bot starts,
- once every 24 hours.

For each coin in `WATCHED_COINS` it:

1. Calls the detailed coin endpoint:

   ```txt
   /coins/{id}?localization=false&tickers=false&community_data=false&developer_data=false&sparkline=false
   ```

2. Extracts from the response:

   - `market_data.ath.usd` → the ATH in USD,
   - `market_data.ath_date.usd` → the date of this ATH (ISO string).

3. Updates the `athState` entry for that coin:

   - `athUsd` = current ATH,
   - `athDate` = ATH date,
   - `athBuffer` = custom buffer (from `WATCHED_COINS`) or default (`0.0005` = 0.05%),
   - `lastAnnouncedAthPrice` = kept from previous run if available, otherwise initialized to `athUsd`.

This data is used later to detect when a new ATH is hit.

---

## 7. `checkPricesAndNotify()` – main monitoring logic

This function is the **core of the bot**. It is called:

- once at startup,
- then repeatedly every `CHECK_INTERVAL_MINUTES`.

### Steps:

1. Calls `fetchSimplePrices()` to get current data for all coins.  
   If it returns `null`, the function exits early.

2. Fetches the Discord channel using `client.channels.fetch(CHANNEL_ID)`.  
   If the channel is not found or is not a text channel, it logs an error and exits.

3. Computes:
   - `todayKey` = current day in UTC (`YYYY-MM-DD`),
   - `now` = current timestamp (ms).

4. For **each coin** in `WATCHED_COINS`:

   #### a) Get current price & 24h change

   - If the API data is missing or does not contain a numeric `usd` price, the coin is skipped.
   - Else:
     - `currentPrice` = `info.usd`,
     - `change24h` = `info.usd_24h_change` (default to 0 when missing).

   #### b) Manage daily opening price

   - Looks up current entry in `dayOpenPrices` for this coin.
   - If there is no entry, or the stored `dayKey` differs from `todayKey` (meaning a new UTC day started):
     - sets a new entry: `{ price: currentPrice, dayKey: todayKey }`,
     - resets `dailyAlertState` to `{ up: false, down: false }` for this coin,
     - logs `[DAYOPEN] Reset ouverture ...`.

   Then, it reads `openPrice` and computes:

   ```js
   const dailyChange = ((currentPrice - openPrice) / openPrice) * 100;
   ```

   → This is the **daily percentage change vs the opening price**.

   #### c) Intraday alert – fast movement over the last interval

   - Gets `lastPrice` from `lastPrices` for this coin.
   - If `lastPrice` exists:
     - computes:

       ```js
       const diffPercent = ((currentPrice - lastPrice) / lastPrice) * 100;
       ```

     - if `|diffPercent| >= coin.intraday` (coin‑specific threshold in %):
       - checks the cooldown (`intradayCooldown`):
         - if the last intraday alert was more than 15 minutes ago:
           - calls `sendIntradayAlert(...)` to send an embed containing:
             - current price,
             - intraday % change over the interval,
             - daily change vs opening price,
             - 24h change,
           - updates `intradayCooldown` to `now`,
           - updates `lastPrices` to `currentPrice`.

   - If `lastPrice` does not exist (first run for this coin):
     - it simply stores `currentPrice` into `lastPrices` and does **not** send an alert.

   #### d) Daily trend alerts – thresholds up/down vs open

   - Gets the state `{ up, down }` from `dailyAlertState` for this coin (or default).
   - If `up` is `false` and `dailyChange >= coin.dailyUp`:
     - calls `sendDailyTrendAlert(...)` with direction `"up"`,
     - marks `state.up = true` and saves it back.

   - If `down` is `false` and `dailyChange <= coin.dailyDown`:
     - calls `sendDailyTrendAlert(...)` with direction `"down"`,
     - marks `state.down = true`.

   This ensures each daily threshold alert (up or down) is sent **at most once per day** per coin.

   #### e) ATH alerts

   - Calls `checkAthAlert(channel, coin, currentPrice, change24h, dailyChange)` for this coin.

---

## 8. Alert sending functions (embeds)

### `sendIntradayAlert(channel, coin, price, diffPercent, dailyChange, change24h)`

Builds a Discord **embed** for **fast intraday moves**:

- Title: “🔼” or “🔻” + coin name/symbol.
- Color: green for upward moves, red for downward moves.
- Fields:
  - current price,
  - intraday change over the interval,
  - daily change vs opening price,
  - 24h change from CoinGecko.

It then sends this embed to the configured channel.

### `sendDailyTrendAlert(channel, coin, price, dailyChange, direction, change24h)`

Builds an embed for **daily trend threshold alerts**:

- Title: `🟢` (up) or `🔴` (down) + coin name/symbol.
- Fields:
  - current price,
  - current daily change,
  - thresholds (`dailyUp` / `dailyDown`),
  - 24h change.

This is triggered when the daily % move crosses your configured upper or lower limits.

---

## 9. `checkAthAlert(...)` – detecting new ATHs

For each coin, this function:

1. Reads `athState[coin.id]`.  
   If there is no ATH data (`!state` or `!state.athUsd`), it does nothing.

2. Extracts:
   - `buffer` = `athBuffer` (e.g. 0.0005 = 0.05%),
   - `athRef` = `athUsd` (official ATH).

3. Checks if the current price is above ATH with buffer:

   ```js
   if (price >= athRef * (1 + buffer)) {
   ```

4. To avoid spamming “new ATH” for tiny incremental ticks, it reads `lastAnnouncedAthPrice`:

   - A new ATH alert is sent only if:

     ```js
     price >= lastAnnounced * 1.003
     ```

     which means **+0.3% above the last announced ATH level**.

5. If conditions are met:

   - Computes gain versus ATH:

     ```js
     const gainVsAth = ((price - athRef) / athRef) * 100;
     ```

   - Builds a **“🚀 New ATH”** embed including:
     - current price,
     - previous ATH,
     - % gain vs previous ATH,
     - daily change,
     - 24h change,
     - previous ATH date (if available).

   - Sends the embed.

   - Updates `state.lastAnnouncedAthPrice = price` in `athState`.

This logic ensures that **major ATH breaks** are highlighted, while avoiding incessant spam when price hovers around the ATH.

---

## 10. Daily OHLC data & summary

### `fetchDailyOHLC(coinId)`

Calls:

```txt
/coins/{coinId}/market_chart?vs_currency=usd&days=1
```

- `data.prices` is a list of `[timestamp, price]` pairs for roughly the last 24 hours.
- The function:
  - sets `open` = first price,
  - sets `close` = last price,
  - iterates over all prices to find `high` and `low`,
  - computes:

    ```js
    const dailyChange = ((close - open) / open) * 100;
    ```

- Returns `{ open, close, high, low, dailyChange }` for that coin.

### `sendDailySummary()`

1. Fetches the Discord channel.
2. For each coin:
   - calls `fetchDailyOHLC(coin.id)`,
   - builds a text string like:

     ```text
     Open:  64,200 $ → Close: 66,500 $ (+3.6 %)
     High: 67,100 $ • Low: 63,800 $
     ```

3. Builds a single embed titled `📊 Daily Recap – YYYY-MM-DD`, adds one field per coin with that text.
4. Sends the embed.
5. Updates `lastSummaryDayKey = todayKey` so it won’t send another recap for the same day.

---

## 11. `scheduleLoops()` – internal scheduling

This function wires up the periodic tasks:

1. **Price monitoring loop** (intraday + daily + ATH):
   - Calls `checkPricesAndNotify()` once immediately.
   - Sets an interval to call it every `CHECK_INTERVAL_MINUTES`.

2. **ATH refresh loop**:
   - Calls `refreshAthForAllCoins()` once immediately.
   - Sets an interval to call it every 24 hours.

3. **Daily summary loop**:
   - Sets an interval to run every minute.
   - Each minute:
     - calculates current UTC day (`todayKey`),
     - checks if a summary was already sent today (`lastSummaryDayKey === todayKey`). If yes, does nothing.
     - gets current local time (`hour`, `min`),
     - if current time is at or past `DAILY_SUMMARY_HOUR:DAILY_SUMMARY_MINUTE` and summary not sent yet:
       - calls `sendDailySummary()`.

This makes the bot fully autonomous once started.

---

## 12. Discord `ready` event & login

At the end of the script:

```js
client.once('ready', () => {
  console.log(`Connecté en tant que ${client.user.tag}`);
  scheduleLoops();
});

client.login(process.env.DISCORD_TOKEN);
```

- When the bot successfully connects to Discord, the `ready` event fires:
  - logs the bot’s name,
  - calls `scheduleLoops()` to start all price checks, ATH refresh, and daily summary scheduling.

- `client.login(DISCORD_TOKEN)` starts the actual connection using your bot token from `.env`.

---

## Summary

In short, your `bot.js` script:

1. Connects to Discord and CoinGecko’s Demo API.  
2. Regularly fetches prices for a set of coins (BTC, ETH, SOL, XRP, DOGE, GMT).  
3. Sends alerts when:
   - there is a **fast move** over a short period (intraday alert),  
   - the **daily move vs opening price** crosses configured thresholds (trend alert),  
   - the price breaks above the **all‑time high** with a buffer (new ATH alert).  
4. Once per day, posts a **daily recap** with Open / High / Low / Close and % change.  
5. Uses in‑memory state and cooldowns to avoid spamming alerts.

This makes **Crypto Watchdog** an always‑on, multi‑coin monitoring bot for Discord that helps you catch big moves, new ATHs, and get a clean end‑of‑day summary.
