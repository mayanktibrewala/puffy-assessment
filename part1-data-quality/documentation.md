A. High-Level Design

Goal: Validate incoming raw event data before it hits production analytics, so broken tracking never silently corrupts revenue dashboards.

Approach:
•	One reusable Python validator class: EventDataQualityValidator
•	Input: pandas DataFrame of raw events (from any day / file).
•	Output: list of DQIssue objects (level = ERROR/WARN, check, message, optional sample rows).
•	Checks grouped into:
1.	Schema & timestamps
2.	Identifiers
3.	Event names & payloads (purchases)
4.	Lightweight anomaly detection (volume & revenue)
You’d run this per batch / partition (e.g. per day) and fail the pipeline on ERRORs, log/notify on WARNs.
 

B. What We Check (and Why)
Area	Check	Why it matters
Schema	Missing expected columns	Breaks downstream models & joins outright.
	Unexpected columns (e.g. new IDs)	Surfaces schema drift so transforms can be updated safely.
Timestamps	timestamp parseable, present	Required for time series, sessions, attribution windows.
Identifiers	client_id vs clientId schema drift	Silent ID rename breaks user-level funnels & attribution.
	% events without any ID	High no-ID share → unattributable sessions / revenue.
Event names	Values outside an expected set	New/renamed events (e.g. checkout_completed) drop from KPIs.
Purchase payload	event_data valid JSON	Prevents runtime errors / missing revenue.
	transaction_id & revenue present	Core keys for revenue, deduplication, and order facts.
	Negative / zero revenue	Catches obvious bugs & test orders.
	Duplicate transaction_id (+ conflicting rev)	Direct cause of double counting or skewed revenue.
Anomalies	Daily event volume outliers	Detects partial outages / bot spikes.
	Daily revenue outliers	Flags sudden drops/spikes even if schema “looks fine”.
 

C. Issues Found in the 14-Day Dataset (23 Feb–8 Mar 2025)

Running the framework on /mnt/data/events_20250223–20250308.csv surfaces:

1.	Identifier schema drift (mid-period)
o	23–26 Feb: client_id populated, clientId all null.
o	27 Feb–8 Mar: clientId populated, client_id all null.
o	Framework flags:
	ERROR: ids.schema.rename (expected client_id, got clientId only for later days).
	WARN: schema.columns.unexpected (clientId not in original schema).

2.	Rising share of events with no identifier
o	Early days (23–26 Feb): ~0.5–1.5% events without any ID.
o	Later days: jumps to 8–16% on several days (e.g. 2 Mar: 15.8%, 8 Mar: 14.3%).
o	Framework flags WARN/ERROR via ids.missing once the global % exceeds configured thresholds.

3.	Event name mismatch vs spec
o	Dataset has: page_viewed, email_filled_on_popup, product_added_to_cart, checkout_started, checkout_completed.
o	No purchase event; purchase-equivalent is checkout_completed.
o	Framework flags WARN: event_name.unknown_values so teams can update mappings rather than silently losing purchases from dashboards expecting purchase.

4.	Zero-revenue orders (likely tests)
o	294 checkout_completed events; 10 have revenue = 0 (3.4%).
o	Framework raises WARN: purchase.revenue.zero_fraction with sample rows.

5.	Duplicate transaction_ids, with conflicting revenue
o	4 duplicate IDs: ORD-20250226-400, ORD-20250227-262, ORD-20250301-176, ORD-20250308-149.
o	3 of them have different revenue values across duplicates (e.g. ORD-20250226-400 = 1524 vs 3199).
o	Framework flags:
	WARN: purchase.transaction_id.duplicates (dedup needed), and
	ERROR: purchase.transaction_id.conflicting_revenue.

These are exactly the kinds of issues that would make a mid-period revenue dashboard look “wrong”, and the framework is generic enough to catch future variants (other renames, payload bugs, anomaly days) without being hard-coded to just this incident.
