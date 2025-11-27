1. Purpose

This framework validates raw event data at ingestion time, before it feeds any dashboards or downstream models. The goals are:

	a. Prevent wrong revenue/funnel numbers from reaching stakeholders.
	b. Catch instrumentation changes (schema drift, ID renames, new event names) early.
	c. Provide a generic, reusable set of checks that will also catch future issues, not just the ones in this 14-day incident.

It is implemented as a Python validator class (EventDataQualityValidator) plus a simple CLI.

2. What We Check, and Why

The checks are grouped into 5 buckets.

	A. Schema & Timestamps
			Check												Why it matters
			All expected columns exist							Missing columns break dbt/SQL models and joins.
			Unexpected columns appear							Early warning for schema drift (e.g. new ID fields).
			timestamp exists and is parseable					Required for daily partitions, sessionization, attribution.

	Reasoning: If schema or timestamps are broken, all downstream logic is untrustworthy. These are hard ERRORs.

	B. Identifiers (User / Device)
			Check												Why it matters
			Presence of client_id / clientId					Detects ID renames that would break user-level metrics.
			% of events with no identifier at all				High “no ID” share → unattributable traffic & revenue.
			Both client_id and clientId together				Signals inconsistent tracking that can confuse transformations.

	Reasoning: Reliable attribution and funnels depend on stable IDs. Large gaps or renames are pipeline-blocking issues.

	C. Event Names & Critical Events
			Check												Why it matters
			Unknown event_name values vs a spec list			Flags new/renamed events that must be mapped (e.g. checkout_completed).
			Critical events present (e.g. page_viewed)			If they disappear, conversion metrics silently collapse.

	Reasoning: Revenue and funnel logic often filter on event_name. Silent changes here are a common cause of broken dashboards.

	D. Purchase / Revenue Payload

	For events that represent purchases (purchase and checkout_completed):

			Check												Why it matters
			event_data parses as JSON / dict					Prevents parse failures and missing revenue.
			transaction_id present								Needed to deduplicate orders and join later.
			revenue present and non-negative					Direct input into revenue reporting.
			Share of zero-revenue purchases						Catches test events / partial integrations.
			Duplicate transaction_ids							Prevents double-counting orders.
			Conflicting revenue per transaction_id				Detects data corruption for specific orders.

	Reasoning: This is where the mid-period revenue bug came from. These checks guarantee that “money events” are structurally sound.

	E. Anomaly Detection (Volume & Revenue)

	Over daily aggregates (when validating multiple days together):

		a. Daily event volume vs rolling median (MAD-based outlier detection).
		b. Daily purchase revenue vs rolling median (same method).
		
	These are WARN-only to avoid alert fatigue.
	Reasoning: Even if schema looks fine, sudden drops or spikes in events or revenue usually indicate broken tracking, tagging, or outages.

3. Issues Identified in the 14-Day Dataset

Applied to the provided 14 days, the framework flagged:

	A. ID rename / drift:
		Early days use client_id, later days use clientId.
		Framework flags id.schema_rename and id.both_id_columns / id.missing.

	B. Rising share of events with no ID:
		Later days show significantly more events with neither client_id nor clientId.
		Framework flags this via id.missing once thresholds are exceeded.

	C. Purchase event name mismatch:
		Spec expects purchase, but raw data uses checkout_completed.
		Framework raises event_name.unknown_values and event_name.missing_critical.

	D. Zero-revenue purchase events:
		A non-trivial fraction of purchase events has revenue = 0.
		Framework warns via purchase.zero_revenue_fraction.

	E. Duplicate transaction IDs with conflicting revenue:
		Some transaction_ids appear multiple times with different revenue amounts.
		Framework warns on purchase.duplicate_transaction_ids and raises an ERROR on purchase.conflicting_revenue.
