1. Overview & Objectives
Goal: Transform raw, validated events into analytics-ready tables that:
1.	Describe user behavior (sessions, devices, pages, actions).
2.	Support marketing attribution with a 7-day lookback, for both:
o	First-click (who introduced the user?), and
o	Last-click (who closed the sale?).
Inputs
•	Raw events (after Part 1 validation), one row per event: client_id / clientId, page_url, referrer, timestamp, event_name, event_data, user_agent
Outputs (core models)
Model	Grain	Purpose
stg_events	Event	Clean, canonical event table (IDs, timestamps, channels, payload).
fct_session_events	Event × Session	Event with attached session_id and session attributes.
fct_sessions	Session	One row per session: start/end, channel, device.
dim_users	User	Aggregated user behavior & revenue.
fct_orders	Order (transaction_id)	One row per deduped order with revenue & order metadata.
fct_marketing_touches	Touchpoint (session entry)	All eligible marketing touches for attribution.
fct_order_attribution	Order × Model	First-click / last-click attribution for each order.
 
2. Methodology & Definitions
2.1 Canonical Events (stg_events)
What we do
•	Normalize identifiers: user_id = coalesce(client_id, clientId).
•	Parse timestamp → event_ts (TIMESTAMP, UTC).
•	Parse event_data JSON → transaction_id, revenue, utm_*.
•	Derive channel using UTM first, then referrer, then “Direct”.
Channel derivation (simplified)
if utm_medium in ('cpc','paid_search')         → 'Paid Search'
elif utm_medium in ('paid_social','social')    → 'Paid Social'
elif utm_medium = 'email'                      → 'Email'
elif utm_source not null                       → 'Other Paid'
elif referrer is null                          → 'Direct'
elif referrer like '%google.%'                 → 'Organic Search'
elif referrer like '%facebook.%'…              → 'Social'
else                                           → 'Referral'
Why: stg_events is the single source of truth for all later models. Any change to ID or channel logic happens here.
 
2.2 Sessionization (fct_sessions & fct_session_events)
Session definition
•	Partition events by user_id, ordered by event_ts.
•	A new session starts when:
o	It’s the first event for that user, or
o	gap(prev_event_ts, event_ts) > 30 minutes.
Implementation
•	Use lag(event_ts) + case to flag new sessions.
•	Compute cumulative sum over the flag → session_seq.
•	session_id = concat(user_id, '_', session_seq).
fct_session_events
•	Event-grain table:
o	Each row from stg_events plus session_id and session_seq.
o	Used for path/funnel analysis at session level.
fct_sessions
•	Session-grain aggregation:
session_id, user_id
session_start_ts, session_end_ts, session_date
event_count
entry_channel, entry_utm_source, entry_utm_medium, entry_utm_campaign
device_type (derived from user_agent: Mobile vs Desktop)
Why 30 minutes?
•	Standard web-analytics heuristic.
•	Works with anonymous traffic, requires no explicit app session_id.
•	Parameterizable later if your business needs longer/shorter sessions.
 
2.3 Users (dim_users)
User definition
•	user_id = stable client/device identifier from stg_events.
•	One row per user_id.
Key fields
•	first_seen_ts, last_seen_ts
•	total_events, active_days
•	Counts of key actions:
o	page_views, adds_to_cart, checkouts_started, purchases
•	total_revenue = sum of revenue for purchase events.
Why: Gives high-level user behavior and revenue contribution, and supports segmentation (e.g. high-value vs new users).
 
2.4 Orders (fct_orders)
Order definition
•	transaction_id from purchase-type events (purchase or checkout_completed).
•	There may be multiple events per transaction_id (retries, page reloads).
Aggregation rule
For each transaction_id:
•	order_ts = max event_ts for that id.
•	user_id = max user_id (should be same across duplicates).
•	revenue = max revenue (avoids under-counting when duplicates exist).
•	Keep order_channel_raw, order_utm_* from the same events.
Result: fct_orders grain is one row per real order:
order_id (=transaction_id), user_id, order_ts, order_date, revenue,
order_channel_raw, order_utm_source, order_utm_medium, order_utm_campaign
Why this rule for duplicates?
•	We saw real duplicates with conflicting revenue in Part 1.
•	Choosing max(revenue) and one row per transaction_id avoids double counting while giving a conservative, deterministic choice.
 
2.5 Marketing Touchpoints (fct_marketing_touches)
Touchpoint definition
•	A touch is the entry event of a session where channel != 'Direct'.
•	Grain: one row per session entry that’s attributable to a marketing channel.
Logic
•	In fct_session_events, find rn = 1 per session_id (first event).
•	Keep only rows where channel is non-null and not Direct.
•	Expose: touch_id, session_id, user_id, touch_ts, touch_date, channel, utm_source, utm_medium, utm_campaign
Why session-entry only?
•	Avoids double-counting within the same visit.
•	Matches “visit-based” analytics mental models (similar to GA’s “session source/medium”).
 
2.6 Attribution (fct_order_attribution)
Business requirement
•	7-day lookback window.
•	Support First-click and Last-click attribution.
Eligible touches
For each order in fct_orders:
•	Find all touches from fct_marketing_touches where:
o	touch.user_id = order.user_id, and
o	order_ts - 7 days ≤ touch_ts ≤ order_ts.
Ranked touches
•	For each order_id, sort the eligible touches by touch_ts:
o	rn_first = 1 for earliest touch.
o	rn_last = 1 for latest touch.
Attribution models
•	First-click: take rn_first = 1, assign 100% of order revenue.
•	Last-click: take rn_last = 1, assign 100% of order revenue.
Output table fct_order_attribution: order_id, user_id, order_ts, revenue, model ('first_click' | 'last_click'), touch_id, session_id, touch_ts, channel, utm_source, utm_medium, utm_campaign
Orders without any eligible touch
•	They simply don’t appear in fct_order_attribution.
•	In dashboards, they can be surfaced separately as Direct/Unknown.
Why this design
•	Simple & interpretable: each order has at most 1 row per model.
•	Higher-order models (e.g. linear, time-decay) can be added later by reusing the same eligible touches set.
 
3. Key Design Decisions & Trade-offs
Area	Decision	Trade-off / Rationale
Session window	30 minutes inactivity per user	Standard, simple; may need tuning for long research behavior.
User grain	user_id = device/client ID	Good for anonymous web; cross-device stitching is out of scope.
Orders	Collapse duplicates by transaction_id, max rev	Prevents double-count; chooses a deterministic value.
Touch unit	Session entry, non-Direct only	Avoids over-counting; matches intuitive “visit-based” view.
Attribution	100% of revenue to one touch (first/last)	Easy for stakeholders; future-proof for multi-touch later.
Models table	Single fct_order_attribution with model column	Simpler querying vs separate tables per model.
 
4. Validation & Reconciliation
We validate the transformation in three ways:
4.1 Structural Tests (per model)
•	stg_events
o	user_id, event_ts, event_name not null.
•	fct_sessions
o	session_id not null, unique per row.
•	fct_orders
o	order_id not null, unique.
o	order_id values exist in stg_events.transaction_id.
(These can be expressed as dbt not_null, unique, relationships tests.)
4.2 Reconciliation Checks
1.	Orders count vs raw purchase events
select count(*)   from stg_events
 where event_name in ('purchase','checkout_completed');

select count(*)   from fct_orders;
•	Expect: fct_orders count ≤ raw purchase count (due to deduping).
2.	Revenue totals
select sum(revenue) from stg_events
 where event_name in ('purchase','checkout_completed');

select sum(revenue) from fct_orders;
•	fct_orders revenue should be close to or slightly below raw revenue. Large gaps → missing revenue fields or bad filters.
3.	Events vs session events
select count(*) from stg_events;
select count(*) from fct_session_events;
•	Should match exactly; any difference means we lost events in sessionization.
4.	Attribution coverage
-- Orders with at least one attribution row (per model)
select count(distinct order_id) from fct_orders;

select count(distinct order_id) from fct_order_attribution
 where model = 'last_click';
•	If coverage suddenly collapses (e.g. 80% → 10%), likely a touchpoint/tracking change.
4.3 Automated Daily Checks
•	Create a daily_metrics table (one row per event_date) with:
o	events, users, orders, revenue
o	pct_missing_user_id
o	attributed_orders, attributed_revenue
•	Run simple sanity rules:
o	Hard fail if orders or revenue are 0 but events are not.
o	Alert if pct_missing_user_id > X%.
o	Alert if attributed revenue << total revenue (indicating attribution break).
This keeps the transformation layer trustworthy and ensures marketing decisions are based on consistent, reconciled data.
