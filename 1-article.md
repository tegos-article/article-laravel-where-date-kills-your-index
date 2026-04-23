# Laravel whereDate() Silently Kills Your Index

![Laravel whereDate() Silently Kills Your Index](assets/poster.jpg)

`whereDate('created_at', $date)` looks clean, but on a big table it quietly drops your index and does a full scan.

## The Problem

Say you want notifications created on a specific day. The obvious call:

```php
UserNotification::query()
    ->whereDate('created_at', '2026-04-23')
    ->get();
```

Laravel generates this SQL:

```sql
SELECT * FROM user_notifications
WHERE DATE(created_at) = '2026-04-23'
```

See `DATE(created_at)`? MySQL has to compute that function for every row before comparing. Your `created_at` index is useless, `EXPLAIN` shows a full table scan:

```plaintext
+----+-------------+--------------------+------+---------+
| id | select_type | table              | type | rows    |
+----+-------------+--------------------+------+---------+
|  1 | SIMPLE      | user_notifications | ALL  | 5000000 |
+----+-------------+--------------------+------+---------+
```

On 5k rows you won't notice. On 5M rows you will.

## The Solution

Compare the column directly against a range:

```php
use Illuminate\Support\Facades\Date;

$start = Date::parse('2026-04-23')->startOfDay();
$end = $start->copy()->addDay();

UserNotification::query()
    ->where('created_at', '>=', $start)
    ->where('created_at', '<', $end)
    ->get();
```

Now the SQL looks like this:

```sql
SELECT * FROM user_notifications
WHERE created_at >= '2026-04-23 00:00:00'
  AND created_at <  '2026-04-24 00:00:00'
```

The column is untouched. MySQL can do a clean range scan on the `created_at` index:

```plaintext
+----+-------------+--------------------+-------+------+
| id | select_type | table              | type  | rows |
+----+-------------+--------------------+-------+------+
|  1 | SIMPLE      | user_notifications | range | 1200 |
+----+-------------+--------------------+-------+------+
```

Half-open range (`>=` start, `<` next day) is the safer form - `endOfDay()` ends at `23:59:59.999999`, and comparing against `23:59:59` can quietly miss the last second.

## Why It Works

This is called **sargability** - "Search ARGument ABLE". A predicate is sargable when the column appears as-is, without a function wrapping it. The moment you write `DATE(col)`, `YEAR(col)`, or `LOWER(col)`, the optimizer can't use a standard B-tree index on `col` anymore.

The same trap applies to `whereDay()`, `whereMonth()`, `whereYear()`, and `whereTime()` all of them wrap the column in a MySQL function. Fine on small lookup tables. Painful on any growing log-style table.

Heads up: PostgreSQL lets you build a functional index (`CREATE INDEX ON t ((date(created_at)))`), so there `whereDate()` can still hit an index. MySQL has no real equivalent for this case, generated columns with an index work, but they're extra schema baggage for something a range filter already solves.

Tutorials love `whereDate('created_at', today())`. I still prefer the range form. It reads the same everywhere, and I never have to wonder whether an index will be used.

## TL;DR

`whereDate()` is convenient but non-sargable on large tables it forces a full scan. Compare `created_at` against a half-open `>= startOfDay()` / `< startOfDay() + 1 day` range and keep the index in play.

💡 Same story for `whereMonth()`, `whereYear()`, `whereDay()`, `whereTime()`. If the column is wrapped in a function, assume the index is gone.

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Notes from real-world Laravel.**