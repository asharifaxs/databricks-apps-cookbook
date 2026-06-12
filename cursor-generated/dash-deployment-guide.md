# Dash App Deployment Guide (Databricks Apps)

> Cursor-generated companion for hands-on deployment.  
> **Model:** Composer 2.5 | **Updated:** 2026-06-11

## Dash folder — what each file does

| File / folder | Purpose |
|---------------|---------|
| `app.py` | **Front door** — creates the Dash app, sidebar navigation, layout shell |
| `app.yaml` | **Runtime contract** — tells Databricks Apps to run `python app.py` |
| `requirements.txt` | **Dependencies** — Python packages installed at deploy time |
| `pages/` | **Recipes** — one file per feature (read table, invoke model, etc.) |
| `pages/book_intro.py` | Landing / introduction page |
| `pages/tables_read.py` | Query Unity Catalog tables via SQL warehouse |
| `assets/` | Logo (`logo.svg`) and styling (`style.css`) |

## Free Edition notes

- **1 SQL warehouse** (2X-Small) — enough for table read recipes
- **Apps auto-stop after 24 hours** — restart anytime from Compute → Apps
- **Lakebase not available** — skip **OLTP Database** recipe
- **Limited model serving / vector search** — AI recipes may need extra setup
- **Serverless only** — **Connect cluster** recipe may not apply

## Post-deploy permissions checklist

Grant the app's **service principal** (Apps → your app → Permissions):

1. **CAN USE** on your SQL warehouse
2. **USE CATALOG** + **USE SCHEMA** + **SELECT** on a test table (e.g. `samples.nyctaxi.trips`)

## First recipes to try (in order)

1. **Authentication → Get current user** — proves Apps identity works
2. **Tables → Read a Delta table** — proves warehouse + UC access works
3. Pick one AI or Workflow recipe once basics work
