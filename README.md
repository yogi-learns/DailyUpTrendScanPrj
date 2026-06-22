# Telegram Stock Scan Script

A Google Apps Script workflow that scans NSE symbols from a Google Sheet, builds historical OHLC blocks with `GOOGLEFINANCE`, calculates EMA and Heiken-Ashi indicators, detects buy signals, and sends the result to Telegram.

## What this script does

The script is built around a simple workflow:

1. Read stock symbols from the `Main` sheet.
2. Build historical price blocks for each symbol in the `HIST` sheet.
3. Compute indicators for each symbol:

   * EMA 8
   * EMA 21
   * EMA 50
   * Heiken-Ashi Open
   * Heiken-Ashi Close
4. Flag a `BUY` signal when the trend and candle pattern match the rule set.
5. Copy the BUY list into a helper sheet and send the results to Telegram.

## Sheets used

### `Main`

Expected input and output sheet.

* **Column A**: NSE symbols, starting at `A2`
* **Columns C:H**: calculated indicators and signal output

Suggested layout:

| Column | Meaning           |
| ------ | ----------------- |
| A      | Symbol            |
| C      | EMA 8             |
| D      | EMA 21            |
| E      | EMA 50            |
| F      | Heiken-Ashi Open  |
| G      | Heiken-Ashi Close |
| H      | Signal            |

### `HIST`

Temporary historical data store.

This sheet is cleared and rebuilt by `populateHistoryBlocks()`. It stores separate OHLC blocks per symbol so the indicator logic can read a stable history range.

### `SendTelegramTable`

Temporary helper sheet.

This sheet is used to hold only the rows where `Signal = BUY` before the final Telegram message is sent.

## Signal logic

A symbol gets a `BUY` signal when all of the following are true:

* `EMA 8 > EMA 21 > EMA 50`
* Today’s Heiken-Ashi candle is bullish
* Yesterday’s Heiken-Ashi candle was bearish

In code, this is the rule:

```javascript
const trendUp = (ema8 > ema21) && (ema21 > ema50);
const haBullToday = haCloseLatest > haOpenLatest;
const haBearYesterday = haClosePrev < haOpenPrev;
const signal = (trendUp && haBullToday && haBearYesterday) ? "BUY" : "NoSignal";
```

## How the scan runs

The main orchestration function is:

```javascript
UpdateScan()
```

It runs these steps in order:

1. `populateHistoryBlocks()`
2. `fillIndicators()`
3. `createBuyListQuery()`
4. `sendBuySignals()`

## Function overview

### `sendTelegram(text)`

Sends a message to a Telegram chat using the Telegram Bot API.

Requires:

* `TG_TOKEN`
* `TG_CHAT_ID`

### `populateHistoryBlocks()`

Creates the historical OHLC blocks inside the `HIST` sheet for every symbol listed in `Main!A2:A`.

What it uses:

* `GOOGLEFINANCE("NSE:<symbol>", ...)`
* About `140` days of history per symbol

### `computeEMA(values, period)`

Calculates the Exponential Moving Average for a list of numeric values.

### `setQueryFormula(targetSheetName, targetRangeA1, sourceSheetName, sourceRangeA1, querySQL, headers)`

Writes a Google Sheets `QUERY` formula into a target cell.

### `clearSheet(sheetName)`

Clears the contents of a sheet.

### `createBuyListQuery()`

Clears `SendTelegramTable` and writes a query that extracts only the rows where `Signal = BUY`.

### `fillIndicators()`

Reads OHLC data from `HIST`, computes the indicators, and writes results back to `Main`.

It also sorts `Main!A2:H` by the signal column.

### `removeBlankRowsAndReturnNewRange(sheetName, rangeA1)`

Removes blank rows from a range and writes the compacted result back to the same sheet.

### `sendTableToTelegram(sheetName, rangeA1)`

Formats the selected range as a simple text table and sends it to Telegram.

### `sendBuySignals()`

Reads the BUY list from `SendTelegramTable`, removes blank rows, and sends the results to Telegram.

### `UpdateScan()`

Runs the full workflow end to end.

## Setup

### 1) Create the Google Sheet

Create a spreadsheet with these sheets:

* `Main`
* `HIST`
* `SendTelegramTable`

### 2) Add symbols to `Main`

Put NSE symbols in `Main!A2:A`, one symbol per row.

Example:

```text
A2: RELIANCE
A3: TCS
A4: INFY
```

### 3) Add the Apps Script

Open **Extensions → Apps Script** and paste the code into a script file.

### 4) Configure Telegram

Replace these placeholders:

```javascript
const TG_TOKEN = "XXXXXX";
const TG_CHAT_ID = "XXXXXX";
```

with your Telegram bot token and chat ID.

### 5) Authorize the script

The script needs authorization for:

* Google Sheets access
* External HTTP requests (`UrlFetchApp`) for Telegram
* `GOOGLEFINANCE` formulas inside the spreadsheet

### 6) Run the scan

Run:

```javascript
UpdateScan()
```

This will rebuild history, calculate indicators, generate the BUY list, and send the output to Telegram.

## Notes and behavior

* The script expects **NSE symbols** and prefixes them with `NSE:` in `GOOGLEFINANCE`.
* `populateHistoryBlocks()` clears the `HIST` sheet before rebuilding it.
* `fillIndicators()` assumes the historical block layout matches what `populateHistoryBlocks()` creates.
* The Telegram alert text uses HTML formatting, so `parse_mode: "HTML"` is enabled.
* The actual Telegram send inside the BUY signal block is currently commented out:

```javascript
//sendTelegram(msg);
```

Uncomment it if you want the script to send an individual alert when a BUY signal is detected.

## Example output

A BUY alert sent to Telegram looks like this:

* Symbol name
* TradingView link
* Timestamp
* Summary of the indicator conditions

The final table message also includes only the symbols that qualify as BUY.

## Common issues

### No symbols appear in `Main`

Make sure symbols are entered in `Main!A2:A` and are plain text values.

### `GOOGLEFINANCE` returns empty values

This can happen if:

* the symbol is invalid
* the market data is temporarily unavailable
* Google Finance does not support the requested field at that moment

### Telegram message is not sent

Check:

* bot token
* chat ID
* whether the bot has permission to message the target chat
* whether the HTTP request is being blocked or failing silently

### Script writes no indicators

Make sure the `HIST` sheet is populated correctly before `fillIndicators()` runs.

## Suggested improvements

* Un-comment the per-symbol Telegram alert if immediate alerts are needed.
* Add validation for missing or invalid OHLC values.
* Write outputs to `Main` with formatting or color coding for signals.
* Add a time-based trigger to run `UpdateScan()` automatically.
* Store `TG_TOKEN` and `TG_CHAT_ID` in Script Properties instead of hardcoding them.

## Security note

Do not commit your Telegram bot token or chat ID into shared source control. Store secrets securely when possible.

## Entry point

Use this function to run everything:

```javascript
UpdateScan()
```
