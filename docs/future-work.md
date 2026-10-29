# Future Work: Portfolio Management Features

Planned portfolio features, grouped by priority.

## Must-have (MVP)

1. **Multiple portfolios**, plus a combined "All portfolios" view.
2. **Fees & charges**: default 1.12% per portfolio pre-fills the fee; each transaction stores its actual fee (editable, e.g. to match the contract note or 0.00 for IPOs). Changing the rate affects future transactions only. Added to buy cost, subtracted from sell proceeds.
3a. **Optional tags** on buys and sells: starts with Investment, IPO and Trade; user can add their own. Labels only, used for filtering (and later P&L by tag).
3. **Dividends**, with two views:
   - *My dividends*: upcoming dividends for held stocks, expected amount from shares held before the XD date, status Announced → XD → Paid.
   - *All dividends*: market-wide upcoming dividends for the next 3 months, plus each company's dividend history.
   - Source: CSE data automatically, plus manual entry.
4. **Corporate actions** (bonus / scrip, splits / subdivisions, rights issues): auto-detected, applied after user confirmation.
5. **Total return per stock**: price profit + dividends − fees.
6. **Portfolio summary dashboard**: total value, invested, today's change, P&L, realized vs unrealized.
7. **Transaction history**: filter by stock and date; undone sells in a separate panel.

## Important (soon after MVP)

8. Performance over time chart.
9. Benchmark comparison vs ASPI and S&P SL20.
10. XIRR (money-weighted) and TWR (time-weighted) returns.
11. Allocation by sector and stock, with concentration warnings.
12. Cash tracking (optional per portfolio).
13. Price alerts.
14. Watchlist.

## Convenience

15. CSV import from broker statements.
16. Export to Excel / PDF.
17. Notes per transaction.
18. Dividend calendar.

## Premium / later

19. Risk metrics: volatility, beta, max drawdown, Sharpe ratio.
20. Forecasts applied to the user's portfolio.
21. What-if simulator.
22. Rebalancing helper (educational).
23. Tax / annual report.
24. Family / shared portfolios.

**Suggested MVP scope:** items 1–7, plus 8, 11 and 14.

---

## Agreed portfolio rules

| Feature | Decision |
|---|---|
| Lots | Every buy is stored as a separate lot |
| Holdings display | Portfolio page: one row per stock using average cost. Lot breakdown only per stock (expand row / stock detail), with realized P&L and dividends. Display never changes any number |
| Selling | Popup to choose which lot(s) to sell from, per sale; oldest lot (FIFO) pre-selected; can split across lots. No "average" sell |
| Realized P&L | Always based on the cost of the chosen lots |
| Undo | **Sells only**, any time in history; undone sells move to an Undone panel with restore |
| Editing buys | Allowed only if it doesn't affect sold shares; otherwise "Undo the related sells first" |
| Data model | Append-only transactions; holdings and P&L calculated in memory by the domain |

## Other planned work

- **Analysis (MVP):** stock page (charts, indicators, key stats), market overview, screener, risk stats.
- **Forecasting (MVP):** 70% range + bias + confidence for 1/5/20 days; baselines; liquidity filter; daily batch after close; public accuracy page; feedback loop (daily adaptive calibration, weekly retraining, champion vs challenger).
- **Knowledge base (phase 2):** RAG Q&A, report summaries, quarter comparisons, with source citations.
- **Later:** news/announcement sentiment features, macro data, live intraday updates (SignalR), mobile app, optional RL research.
