1. Goal & Approach

  Goal: Turn validated raw events into an analytics layer that answers:
  a. User engagement – how people browse, from where, on which devices, and what they do.
  b. Marketing performance – which channels get credit for revenue (first-click & last-click) with a 7-day lookback.

 Approach: Use a small set of clear-grain models, with a clean separation between:
  a. Staging (parsing/normalization),
  b. Core facts/dimensions (sessions, users, orders),
  c. Marketing attribution (touchpoints, attribution results).

2. Architecture & Core Models

 2.1 Model Overview
     
     Model                 	Grain                        	Main Use
     stg_events	            Event	                        Clean canonical events (IDs, timestamps, JSON, channel, device).
     fct_session_events	    Event × Session	              Event-level funnels & paths with session_id.
     fct_sessions	          Session	                      Session-level behavior (entry channel, device, counts).
     dim_users	             User (user_id)	               Aggregated user activity & revenue.
     fct_orders	            Order (transaction_id)	       Deduped orders with revenue & order_ts.
     fct_marketing_touches 	Touchpoint (session entry)	   Eligible marketing touches for attribution.
     fct_order_attribution	 Order × model	                First-click / last-click attribution (7-day window).

 2.2 stg_events (Canonical Events)

   From raw_events_validated:
     
     A. User ID:
     user_id = coalesce(client_id, "clientId") → Handles the historical client_id → clientId change.

     B. Timestamps:
     event_ts = "timestamp"::timestamptz, event_date = event_ts::date.

     C. Purchase fields (JSON):
     transaction_id, revenue from event_data.

     D. UTM fields (JSON):
     utm_source, utm_medium, utm_campaign.

     E. Channel:
     Priority: UTM → referrer → Direct, giving buckets like: Paid Search, Paid Social, Email, Other Paid, Organic Search, Social, Referral, Direct.

     F. Device:
     device_type = 'Mobile' | 'Desktop' | 'Unknown' from user_agent.

3. Definitions – Sessions, Users, Attribution

 3.1 Sessions

   Rule: 30-minute inactivity window per user_id.
     a. Sort events by event_ts per user_id.
     b. New session if:
        i. first event, or
        ii. gap from previous event > 30 minutes.
     c. session_seq = cumulative sum of "new session" flags.
     d. session_id = user_id || '-' || session_seq.

   fct_session_events (event grain)
   All columns from stg_events + session_id & session_seq.

   Used for: page → cart → checkout → purchase funnels, step conversion within a visit.

   fct_sessions (session grain)
   Per session_id:

   a. session_start_ts, session_end_ts, session_date, event_count.
   b. Entry attributes from first event:
      i. entry_event_name, entry_page_url,
      ii. entry_channel, entry_utm_source/medium/campaign,
      iii. device_type.

  Used for: “What did a visit look like? Where did it come from? On which device?”

3.2 Users – dim_users

User: stable user_id from stg_events.
Aggregations (from fct_session_events):
  a. first_seen_ts, last_seen_ts,
  b. session_count, total_events,
  c. pageviews, adds_to_cart, checkouts_started,
  d. purchases (purchase-like events),
  e. total_revenue (sum of purchase revenue).

Answers: “How active and valuable are our users?” and supports cohorts/segmentation.

3.3 Orders – fct_orders

 Order grain: transaction_id.

 From stg_events where event_name in ('purchase','checkout_completed') and transaction_id is not null.

 Because we observed duplicate transaction IDs with conflicting revenue, we dedupe:

 Per transaction_id:
  a. order_id = transaction_id
  b. user_id = max(user_id)
  c. order_ts = max(event_ts) (latest)
  d. order_date = max(event_date)
  e. revenue = max(revenue) (conservative, deterministic)
  f. order_channel, order_utm_* = max(...) from grouped rows.
 This gives one row per real order and avoids double-counting.

3.4 Marketing Touchpoints – fct_marketing_touches

 Touchpoint: a non-direct session entry.

 From fct_sessions:

  a. Keep sessions with entry_channel <> 'Direct'.
  b. For each:
     i. touch_id (here = session_id),
     ii. session_id, user_id,
     iii. touch_ts = session_start_ts, touch_date = session_date,
     iv. channel, utm_source, utm_medium, utm_campaign.

  This defines clean marketing touches (one per marketing-driven visit), suitable for attribution.

3.5 Attribution – fct_order_attribution

  Constraint: 7-day lookback.

  For each order in fct_orders:

  Eligible touches:
  a. Join fct_orders to fct_marketing_touches on user_id.
  b. Keep touches where: touch_ts between (order_ts - 7 days) and order_ts

  Rank touches (per order):
  a. rn_first = earliest touch_ts.
  b. rn_last = latest touch_ts.

  Build output:
  a. First-click: rows where rn_first = 1, model = 'first_click'.
  b. Last-click: rows where rn_last = 1, model = 'last_click'.

  Columns:
  a. model, order_id, user_id, order_ts, order_date, revenue,
  b. touch_id, session_id, touch_ts,
  c. channel, utm_source, utm_medium, utm_campaign.

  Each model assigns 100% of order revenue to a single touch. Multi-touch models can reuse the same eligible touches.

4. Key Design Decisions & Trade-offs

       Area                  	Decision	                                       Trade-off / Why it’s reasonable
       Session window	        30-minute inactivity	                           Standard, intuitive; tunable if product behavior changes.
       User ID	               coalesce(client_id, "clientId")	                Handles historic ID rename; can extend to more IDs later.
       Orders	                Group by transaction_id, max(revenue)          	Avoids double-count; picks a single, consistent revenue per order.
       Touch grain	           Session-entry, non-direct only	                 Avoids intra-session over-count; matches visit-based analytics.
       Attribution	           Single-touch first/last-click, 7-day window	    Simple, understandable; provides both “who discovered” and “who closed” views.
       Layering	              Staging → core facts → attribution layer	       Keeps models small, testable and easy to maintain/scale.
