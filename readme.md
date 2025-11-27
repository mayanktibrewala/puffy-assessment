# Event Pipeline – Data Quality, Transformation, Analysis & Monitoring

This repo contains a full mini-analytics stack built on a 14-day web events dataset:

- **Part 1 – Incoming Data Quality**
- **Part 2 – Transformation Pipeline**
- **Part 3 – Business Analysis**
- **Part 4 – Production Monitoring**

Each part has:
- `documentation.md` – design / explanation
- `code-files/` (or `code files`) – code / SQL
- Plus `executive-summary.pdf` for Part 3

---

## 1. Repository Layout

```text
.
├─ part1-data-quality/
│   ├─ code-files/             # Python data quality framework
│   └─ documentation.md
├─ part2-transformation/
│   ├─ code-files/             # SQL models for transformations
│   ├─ documentation.md
│   └─ validation/             # validation notes / queries
├─ part3-analysis/
│   └─ executive-summary.pdf   # business analysis for management
├─ part4-monitoring/
│   ├─ code-files/             # Python monitoring script(s)
│   └─ documentation.md
└─ readme.md


Setup instructions

a. Install Python deps
b. Load raw events into your warehouse as raw_events_validated.
c. Create a daily_metrics table/view from the transformed models.
d. Set DB URL for monitoring: export ANALYTICS_DB_URL=postgresql+psycopg2://user:pass@host:5432/db


How to run my code

# 1) Data quality on CSVs
python part1-data-quality/code-files/part1_data_quality.py --data-dir ./data

# 2) Run SQL models in part2-transformation/code-files in your database (or via dbt)

# 3) Daily monitoring
python part4-monitoring/code-files/monitor_daily_metrics.py
