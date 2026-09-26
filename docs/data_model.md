# Data Model

## Tables

- `Stores` — store dimension
- `Categories` — category dimension
- `Sales` — sales fact
- `Targets` — target fact
- `DateTable` — calendar dimension

## Relationships

- DateTable[Date] → Sales[sale_month]
- DateTable[Date] → Targets[target_month]
- Stores[store_id] → Sales[store_id]
- Stores[store_id] → Targets[store_id]
- Categories[category_id] → Sales[category_id]

The model uses dimension-to-fact filtering with a dedicated date table.
