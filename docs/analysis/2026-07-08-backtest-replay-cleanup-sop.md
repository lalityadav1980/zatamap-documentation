# Backtest Replay Cleanup SOP

Date: 2026-07-08

## Rule

Before restarting any single-day backtest replay, clear all prior backtest users for that replay date. Do not clear paper or live users.

For a replay date `YYYY-MM-DD`, the cleanup scope is:

- `trade.orders_book.zerodha_id LIKE 'bt_YYYYMMDD%'`
- `trade.orders_tracker.zerodha_id LIKE 'bt_YYYYMMDD%'`
- `trade.fusion_events.user_id LIKE 'bt_YYYYMMDD%'`
- `trade.trail_data` rows whose `parent_order_id` belongs to those backtest orders or trackers
- `trade.trail_summary_day` rows whose `parent_order_id` belongs to those backtest orders or trackers

Paper/live users such as `<user_id>` must never be touched by this replay cleanup.

## Why

Scoped cleanup by only the latest diagnostic user is technically safe, but it leaves old `bt_YYYYMMDD_*` rows visible in the UI and makes replay validation ambiguous. A restarted replay must have exactly one fresh data source for that date, so UI, SQL, logs, and fusion events all describe the same run.

## SQL Template

```sql
BEGIN;

CREATE TEMP TABLE _bt_parent_ids AS
SELECT DISTINCT parent_order_id
FROM trade.orders_book
WHERE zerodha_id LIKE 'bt_YYYYMMDD%'
  AND parent_order_id IS NOT NULL
UNION
SELECT DISTINCT parent_order_id
FROM trade.orders_tracker
WHERE zerodha_id LIKE 'bt_YYYYMMDD%'
  AND parent_order_id IS NOT NULL;

DELETE FROM trade.trail_data
WHERE parent_order_id IN (SELECT parent_order_id FROM _bt_parent_ids);

DELETE FROM trade.trail_summary_day
WHERE parent_order_id IN (SELECT parent_order_id FROM _bt_parent_ids);

DELETE FROM trade.orders_tracker
WHERE zerodha_id LIKE 'bt_YYYYMMDD%';

DELETE FROM trade.fusion_events
WHERE user_id LIKE 'bt_YYYYMMDD%';

DELETE FROM trade.orders_book
WHERE zerodha_id LIKE 'bt_YYYYMMDD%';

COMMIT;
```

After cleanup, verify all five counts are zero before launching the replay.
