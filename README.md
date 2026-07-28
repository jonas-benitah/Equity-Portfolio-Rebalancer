# Equity Portfolio Rebalancer

An n8n workflow that reads a stock portfolio from Google Sheets, fetches live market prices, computes the rebalancing needed to reach a target allocation, writes the results back to the sheet, and notifies the user by email and push notification.

> Built with n8n, OpenAI (tool-calling agent), Marketstack, Google Sheets, Gmail and Pushover.

---

## Why this project

Rebalancing a portfolio is a task that is simple in principle but tedious in practice: pull current prices, recompute the equity/fixed-income split, decide how many units to buy or sell, record the result. This workflow automates the full loop and delivers the trade list without any manual step.

It was built as a hands-on exercise in **multi-service API orchestration** — the interesting part is not the finance, it is making six external services cooperate reliably in a single run.

---

## Architecture

```
Form submission (trigger)
        │
        ▼
   AI Agent (OpenAI, tool-calling)
        │
        ├── Google Sheets — read portfolio
        ├── Marketstack   — fetch end-of-day prices
        ├── Google Sheets — write updated prices
        ├── Google Sheets — write new quantities
        ├── Pushover      — push notification
        └── Gmail         — send trade summary
```

The agent receives the tools above and decides which to call in which order, based on the instructions in its system prompt.

---

## Stack

| Layer | Service |
|---|---|
| Orchestration | n8n (self-hosted) |
| Reasoning | OpenAI (chat model + tool calling) |
| Market data | Marketstack (end-of-day prices) |
| Data store | Google Sheets |
| Notifications | Gmail, Pushover |

---

## Data model

The Google Sheet holds one row per ticker:

| Column | Field | Source |
|---|---|---|
| A | Ticker | manual |
| B | Quantity | manual |
| C | Equity ratio | manual |
| D | Fixed income ratio | manual |
| E | Price | written by workflow |
| F | Total Value | formula |
| G | New Quantity After Rebalancing | written by workflow |
| H | New Total Value | formula |

Row 6 totals the portfolio; rows 7–8 compute the current equity / fixed-income split:

```
F7 = SUMPRODUCT(F2:F5, C2:C5) / SUM(F2:F5)
F8 = SUMPRODUCT(F2:F5, D2:D5) / SUM(F2:F5)
```

Each holding carries its own equity/fixed-income weighting, so blended ETFs (e.g. AOR at 60/40) are handled correctly rather than being forced into a single bucket.

---

## How it runs

1. The user submits the n8n form, giving the target allocation.
2. The agent reads every row of the portfolio from the sheet.
3. It requests the latest end-of-day price for each ticker from Marketstack.
4. Prices are written back to column E; the sheet formulas recompute total value and current split.
5. The agent computes the quantities needed to reach the target allocation and writes them to column G.
6. A summary of the trades is emailed via Gmail, and a push notification is sent via Pushover.

---

## Setup

**Requirements:** an n8n instance, a Google account, an OpenAI API key, a Marketstack API key, a Pushover account.

1. Import `workflow.json` into n8n.
2. Create the credentials: Google Sheets (OAuth2), Gmail (OAuth2), OpenAI, Marketstack, Pushover.
3. Copy the spreadsheet template and set its ID in the Google Sheets nodes.
4. **Set the spreadsheet locale to United States** (File → Settings → Locale). With a French locale the decimal separator is a comma, which breaks both the formulas and the numeric values written by the API.
5. Fill columns A–D with your holdings, then publish the workflow.

---

## Known limitations

Stated plainly, since these are the boundaries of the current version:

- **End-of-day prices only.** Marketstack's free tier does not provide intraday data, so the rebalancing reflects the previous close.
- **Whole units only.** Fractional shares are not handled, which means the resulting allocation lands close to the target rather than exactly on it.
- **No missing-price handling.** If Marketstack returns nothing for a ticker, the price cell stays at 0 and the percentage cells return `#DIV/0!`.
- **No transaction costs.** Broker fees and spread are ignored.
- **Nothing is executed.** The workflow produces a trade list; placing the orders remains manual, which is deliberate.

## Next steps

- Drift threshold: skip rebalancing entirely when the deviation from target stays under a configurable percentage, to avoid churning on noise.
- Retry and fallback on the price API, with an explicit error notification instead of a silent zero.
- An execution history tab logging date, total value and allocation after each run.

---

## Notes

Originally built as a guided project, then extended and documented independently. The parts I would defend in a technical discussion are the data model (per-holding weightings rather than fixed buckets), the choice to keep order execution manual, and the locale constraint described above — each was a decision, not a default.

**Author:** Jonas Benitah — [GitHub](https://github.com/jonas-benitah)
