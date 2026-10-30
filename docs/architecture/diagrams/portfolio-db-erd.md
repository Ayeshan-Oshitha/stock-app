# portfolio_db ER Diagram

Tables and relationships for `portfolio_db`. Column details: [../portfolio-db.md](../portfolio-db.md).

Grey tables and columns marked `later` are added later (corporate actions, dividends, events). Types are simplified here (`decimal` = `numeric(18,2)` or `numeric(7,4)`); exact types are in the column tables.

```mermaid
erDiagram
    companies ||--o{ holdings : "is held in"
    companies ||--o| latest_prices : "has"
    portfolios ||--o{ holdings : "contains"
    holdings ||--o{ transactions : "has"
    transactions ||--o{ sell_allocations : "sell takes from"
    transactions ||--o{ sell_allocations : "buy (lot) gives to"
    transactions ||--o{ sell_status_history : "undo / restore"
    transactions ||--o{ transaction_tags : "labelled with"
    tags ||--o{ transaction_tags : "used in"
    transactions ||--o{ lot_adjustments : "lot changed by"
    holdings ||--o{ dividend_entitlements : "entitled to"

    companies {
        uuid id PK "same id as Market Data"
        varchar symbol UK
        varchar name
        varchar sector
        varchar status "Active, Suspended, Delisted"
        timestamptz updated_at
        varchar share_type "later"
        timestamptz source_updated_at "later"
    }
    latest_prices {
        uuid company_id PK, FK
        decimal price
        date price_date
        varchar source "Manual, MarketData"
        timestamptz updated_at
        uuid updated_by
        decimal previous_close "later"
    }
    portfolios {
        uuid id PK
        uuid user_id "Keycloak sub"
        varchar name
        decimal default_fee_rate "default 1.12"
        timestamptz created_at
        timestamptz updated_at
        timestamptz archived_at
        varchar description "later"
        int display_order "later"
    }
    holdings {
        uuid id PK
        uuid portfolio_id FK
        uuid company_id FK
        uuid user_id
        timestamptz created_at
        timestamptz updated_at "changed on every change"
    }
    transactions {
        uuid id PK "buy id = lot id"
        uuid holding_id FK
        uuid user_id
        varchar type "Buy, Sell"
        varchar status "Active, Undone, Voided"
        date trade_date
        bigint quantity
        decimal price
        decimal fee_rate
        decimal fee_amount
        varchar note
        timestamptz created_at
        timestamptz updated_at
        timestamptz voided_at
        varchar source "later"
        uuid corporate_action_id "later"
        date settlement_date "later"
        varchar contract_note_no "later"
    }
    sell_allocations {
        uuid id PK
        uuid sell_id FK
        uuid buy_id FK
        bigint quantity
    }
    sell_status_history {
        uuid id PK
        uuid sell_id FK
        varchar action "Undo, Restore"
        timestamptz occurred_at
        uuid user_id
        varchar reason "later"
    }
    tags {
        uuid id PK
        uuid user_id "empty = built-in tag"
        varchar name
        timestamptz created_at
        varchar color "later"
    }
    transaction_tags {
        uuid transaction_id PK, FK
        uuid tag_id PK, FK
    }
    lot_adjustments {
        uuid id PK
        uuid buy_id FK
        uuid corporate_action_id
        varchar type "Bonus, Split, Consolidation"
        bigint quantity_change
        decimal fraction_dropped
        date effective_date
        timestamptz confirmed_at
        uuid confirmed_by
    }
    dividend_entitlements {
        uuid id PK
        uuid holding_id FK
        uuid user_id
        uuid dividend_id
        date xd_date
        date payment_date
        bigint quantity_entitled
        decimal amount_per_share
        decimal gross_amount
        decimal tax_amount
        decimal net_amount
        varchar status "Expected, Received"
        date received_on
    }
    outbox_messages {
        uuid id PK
        varchar type
        jsonb payload
        timestamptz occurred_at
        timestamptz processed_at
        int attempts
        text last_error
    }
    inbox_messages {
        uuid message_id PK
        varchar type
        timestamptz processed_at
    }

    classDef later fill:#e5e7eb,stroke:#9ca3af,color:#6b7280
    class lot_adjustments,dividend_entitlements,outbox_messages,inbox_messages later
```
