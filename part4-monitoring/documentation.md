1. Objective

  This pipeline runs daily and powers revenue and marketing dashboards. Monitoring must:
     i. Catch data issues before dashboards refresh.
     ii. Focus on business-critical metrics (traffic, orders, revenue, attribution, funnel).
     iii. Be simple enough to run as the final step in the DAG and fail fast when data is untrustworthy.

  I monitor 3 layers:
     i. Completeness & pipeline health – is the latest day present and populated?
     ii. Data quality & relationships – are keys, joins, and attribution consistent?
     iii. Business sanity – do core metrics behave “normally” vs the last few days?

2. What I Monitor (and Why)

  Assumption: we have a daily aggregate table daily_metrics with one row per event_date that includes:
    a. events, users, sessions, orders, revenue
    b. Funnel counts: add_to_cart_users, checkout_users, purchase_users
    c. Attribution: attributed_orders_first_click, attributed_revenue_first_click, attributed_orders_last_click, attributed_revenue_last_click

  2.1 Completeness & Volume
    Per day (especially “yesterday”):
    a. Row exists for yesterday in daily_metrics
    → Detects missing partition or DAG failure.
    b. events, users, sessions, orders, revenue are non-null and non-negative
    → Basic sanity: no broken aggregation or type issues.
    c. orders > 0 when traffic is non-trivial
    → If events look normal but orders = 0, something is wrong with purchase tracking or transformation.

    These are hard requirements; if they fail, the job should be treated as failed.

  2.2 Data Quality & Attribution Integrity
  On top of transformation-level tests from Part 2 (unique order_id, unique session_id, etc.), at the daily metric level I check:
    a. Reconciliation:
       i. orders in daily_metrics = count(*) from fct_orders for that date.
       ii. revenue in daily_metrics = sum(revenue) from fct_orders.
    → Ensures the daily aggregate matches the fact table.
    b. Attribution sanity:
    For each model (first_click, last_click):
       i. attributed_orders_* <= orders
       ii. attributed_revenue_* <= revenue
       iii. attributed_orders_* / orders reasonably > 0 (e.g. not suddenly ~0%)
    → Prevents over-attribution and catches broken joins / window logic when coverage collapses.

  2.3 Funnel & Conversion
  From daily_metrics:
    a. Add-to-cart rate = add_to_cart_users / users
    b. Checkout start rate = checkout_users / add_to_cart_users
    c. Purchase (checkout → purchase) rate = purchase_users / checkout_users

  These monitor core funnel health:
    a. A large drop in add-to-cart rate suggests product/UX event issues.
    b. A drop in checkout start rate suggests cart → checkout transition issues.
    c. A drop in purchase rate suggests payment / checkout errors.

If any of these move dramatically while traffic is stable, it’s likely a real site or tracking problem, not random noise.

3. How I Detect Issues

I combine hard rules (for clear failures) and robust statistical checks (for anomalies). The monitoring script pulls the last 8 days of daily_metrics (7-day baseline + yesterday).

  3.1 Hard Rules (Critical)
  These conditions almost always mean “do not trust yesterday’s data”:
    a. No daily_metrics row for yesterday.
    b. Any of events, users, sessions, orders, revenue is NULL or < 0.
    c. orders = 0 while events is within a normal range (pipeline suggests people visited, but nobody bought).
    d. Attribution totals larger than base metrics (e.g. attributed_revenue_* > revenue).

  When any of these fire, the script logs a [CRITICAL] message and exits with non-zero status → scheduler marks the run as failed.

  3.2 Statistical Anomaly Detection (Median + MAD)
  For continuous metrics (volume and conversion rates):
    a. Metrics:
       i. Volume: events, users, sessions, orders, revenue
       ii. Conversion: add-to-cart rate, checkout rate, purchase rate
    b. Method:
       i. Take the previous 7 days as baseline.
       ii. For each metric:
           A. Compute median and MAD (median absolute deviation).
           B. Compute z = (today - median) / MAD (if MAD > 0).
       iii. Thresholds:
           A. If today < 0.3 * median and z <= -5 → CRITICAL (sharp drop).
           B. If today < 0.6 * median and z <= -3 → WARN (significant drop, but not catastrophic).

   This approach:
    a. Uses robust statistics (median/MAD), which are not distorted by occasional spikes. 
    b. Focuses on drops (we care more if revenue goes missing than if it unexpectedly spikes).
    c. Keeps thresholds high enough to avoid alert fatigue from minor fluctuations.

  3.3 Attribution Coverage Checks
  For each model (first_click, last_click):
    a. Coverage = attributed_orders_* / orders (if orders > 0).
  Rules:
    a. If attributed_orders_* > orders or attributed_revenue_* > revenue → CRITICAL.
    b. If orders > 0 and coverage < 10% (e.g. only a tiny fraction of orders attributed) → WARN.
  Reason: if coverage suddenly drops from a normal level to near-zero, it likely means:
    a. Attribution joins broke (e.g. user_id mismatch),
    b. Touch table (fct_marketing_touches) is empty,
    c. Or the 7-day lookback logic malfunctioned.

4. Practical Daily Operation

Where it sits in the DAG:
  a. Raw → Validated → Core models (stg_events, fct_session_events, fct_sessions, fct_orders, fct_order_attribution).
  b. daily_metrics is built.
  c. Final step: monitor_daily_metrics.py runs.

Behavior:
  a. Script prints a human-readable report like:
     i. [OK] events: within normal range (...)
     ii. [WARN] checkout_rate: 0.12 vs median 0.19 (z=-3.1, frac=0.63)
     iii. [CRITICAL] orders: 2 vs median 21 (z=-5.5, frac=0.10).
  b. If any CRITICAL issue is found: exit code 1 → pipeline fails, dashboards can be blocked or annotated.
  c. WARN issues still allow success but should page/notify the data owner; they highlight potential problems to investigate.

This design keeps monitoring:
  a. Focused on the metrics that matter to the business,
  b. Quantitative, not subjective,
  c. And easy to maintain as the data model evolves (add new metrics to daily_metrics and extend the checks if needed).
