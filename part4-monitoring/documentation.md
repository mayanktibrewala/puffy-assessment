1. What I’d Monitor (and Why)
Think of monitoring in three layers. The idea is: Is the data there? Is it valid? Does it make business sense today?
1.1 Layer 1 – Pipeline & Ingestion Health
Goal: Catch “no/partial data” before anyone opens a dashboard.
Area	Metric (daily)	Why it matters
Ingestion	Row count in raw.events (today)	Detects connectors/batch failures.
Transforms	Row counts in stg_events, fct_orders	Ensures core models actually materialised.
Freshness	Max event_date in each core table	Ensures we’re not stuck on old partitions.
Rules:
•	CRITICAL if:
o	raw.events or stg_events has 0 rows for today.
o	Latest partition date < today - 1 day.
•	WARN if:
o	Count < 50% of 7-day median (e.g., traffic suddenly halves with no known cause).
 
1.2 Layer 2 – Data Quality / Transform Integrity
Goal: Guarantee that key entities (users, orders, revenue, attribution) are structurally sound.
Key checks (daily, per event_date):
1.	Identifiers & schema
o	% missing user_id in stg_events.
o	Presence of event_name values like page_viewed, checkout_started, purchase/checkout_completed.
o	Why: prevents silent breaks like client_id → clientId renames and missing conversion events.
2.	Orders & revenue
o	orders in fct_orders.
o	Sum of revenue in fct_orders.
o	duplicate order_id in fct_orders (should be 0).
o	% of orders with revenue = 0orNULL`.
o	Why: these directly drive revenue dashboards.
3.	Session mapping
o	Count in stg_events vs fct_session_events (should match).
o	Why: ensures every event is sessionised; funnels aren’t based on a subset.
4.	Attribution sanity
o	attributed_orders and attributed_revenue vs total orders/revenue.
o	Why: if attribution suddenly covers only 5% of orders, marketing dashboards are useless.
Rules:
•	CRITICAL:
o	% missing user_id > 10%.
o	Any duplicate order_id in fct_orders.
o	0 purchase/checkout_completed events but non-zero traffic.
•	WARN:
o	% missing user_id > 5%.
o	% zero-revenue orders > 5%.
o	Attributed revenue < 30% of total revenue (sudden drop vs norms).
 
1.3 Layer 3 – Business KPI Monitoring
Goal: Metrics might “look valid” but be wrong (e.g., one event stops firing only on mobile).
Daily KPIs (from a daily_metrics aggregate table):
•	Volume: events, users, sessions, pageviews.
•	Conversion funnel:
o	add_to_cart_rate = add_to_cart / pageviews
o	checkout_start_rate = checkout_started / add_to_cart
o	purchase_rate = orders / checkout_started
o	Overall conversion = orders / pageviews.
•	Revenue: orders, revenue, AOV = revenue / orders.
•	By device (mobile vs desktop) and by channel (for revenue/orders).
Detection method:
Simple, robust anomaly detection on the last 7 days as baseline:
•	For each metric M (e.g., orders, revenue, conversion):
o	Compute median m and median absolute deviation (MAD) over previous 7 days.
o	Today’s z-like score: z = (today − m) / MAD.
•	Trigger:
o	CRITICAL if:
	Today’s value < 20% of baseline median, or
	|z| ≥ 6 (strong outlier).
o	WARN if:
	Today’s value < 50% of baseline, or
	|z| ≥ 4.
Examples:
•	Orders tank from ~20/day to 1 with normal traffic ⇒ CRITICAL.
•	Conversion slips from 1.5% to 0.9% with no marketing campaign changes ⇒ WARN, investigate (UX, pricing, tech).
 
2. Detection & Alerting Flow
1.	ETL / dbt job runs, materialises:
o	stg_events, fct_session_events, fct_orders, fct_order_attribution, and daily_metrics.
2.	Monitoring job runs as the final step:
o	Queries daily_metrics for the last 8 days (7-day baseline + today).
o	Applies the rule set above.
3.	Results:
o	If any CRITICAL issue → non-zero exit code → ETL marked failed → Slack/Email alert.
o	If only WARN → job passes but posts a Slack summary or logs to an observability dashboard.
This avoids both silent failures and alert fatigue.
