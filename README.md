# Laravel whereDate() Silently Kills Your Index

![Laravel whereDate() Silently Kills Your Index](assets/poster.jpg)

`whereDate('created_at', $date)` feels like the clean way to filter by day in Laravel.
But it wraps the column in `DATE()`, which makes the predicate **non-sargable** — MySQL can't use your index anymore, and you quietly end up with a full table scan.

This tip shows:

- The SQL Laravel actually generates for `whereDate()`
- Why the index on `created_at` stops working
- A sargable range alternative with `startOfDay()` / `endOfDay()`
- Which other Laravel helpers have the same problem (`whereMonth`, `whereYear`, `whereDay`, `whereTime`)

**Takeaway:**
If a column is wrapped in a function inside `WHERE`, assume the index is gone.

## 📎 Read the Full Article

[Laravel whereDate() Silently Kills Your Index](https://dev.to/tegos/PLACEHOLDER)
