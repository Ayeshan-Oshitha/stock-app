# Portfolio Service (.NET)

Portfolios, lots, buys, sells (lot selection per sale), undo of sells, dividends received, fees and corporate-action adjustments.
Main aggregate: `Holding` (one portfolio + one stock). Transactions are append-only; the domain calculates holdings and P&L in memory. Owns `portfolio_db`.
