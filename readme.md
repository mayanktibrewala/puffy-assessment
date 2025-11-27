# Analytics Pipeline – README

This repository contains a small, end-to-end analytics stack for web/e-commerce events:

1. **Part 1 – Incoming Data Quality Framework** (Python)
2. **Part 2 – Transformation Pipeline** (SQL / dbt)
3. **Part 4 – Production Monitoring** (Python)

Part 3 (Business Analysis) is a written executive summary only and does not contain code.

---

## 1. Repository Structure

Suggested layout (you can adjust names/paths if needed):

```text
.
├─ README.md
├─ requirements.txt
├─ scripts/
│  ├─ validate_events.py          # Part 1 – data quality validator
│  └─ monitor_daily_metrics.py    # Part 4 – production monitoring
└─ dbt_project/
   ├─ dbt_project.yml
   ├─ models/
   │  ├─ stg_events.sql
   │  ├─ fct_session_events.sql
   │  ├─ fct_sessions.sql
   │  ├─ dim_users.sql
   │  ├─ fct_orders.sql
   │  ├─ fct_marketing_touches.sql
   │  ├─ fct_order_attribution.sql
   │  └─ daily_metrics.sql
   └─ profiles.yml (local or in ~/.dbt/profiles.yml)




