# GPay Review Sentiment Analysis Pipeline

An end-to-end data pipeline that scrapes Google Pay reviews from the Play Store, runs them through an NLP sentiment and aspect-classification pipeline, warehouses the results in a PostgreSQL star schema, and visualizes the findings in an interactive 3-page Power BI dashboard.

## Overview

This project analyzes ~16,000 Google Pay reviews to understand what users are actually complaining about and how severe those complaints are. Each review is classified into one of 6 business-relevant categories (Transactions, Security, Rewards, Credit, App Performance, or General Feedback) and scored for sentiment using pre-trained Hugging Face transformer models. The processed data is normalized into a Kimball-style star schema, with four analytical SQL views computing severity scores, rolling sentiment trends, and NLP validation metrics using CTEs and window functions. The final output is a branded, interactive Power BI dashboard supporting drill-through, drill-down, custom tooltips, and dynamic DAX measures.

## Pipeline

<img width="1024" height="1536" alt="ChatGPT Image Sep 6, 2026, 12_31_51 AM" src="https://github.com/user-attachments/assets/0bd29cf8-dd70-48fa-af42-fcc90fbda2b9" />

1. **Ingestion** — `google-play-scraper` pulls reviews (text, rating, app version, date) for the Google Pay app.
2. **Staging** — Raw review data is preserved as-is before any processing, for audit purposes.
3. **NLP processing** — Each review passes through two Hugging Face models: a zero-shot classifier assigning one of 6 fixed aspect categories (with a confidence score), and a sentiment classifier scoring tone from -1 to +1. Reviews below a confidence threshold default to "General Feedback."
4. **Data warehousing** — Results are normalized into a star schema in PostgreSQL (hosted on Supabase): `dim_review`, `dim_aspect`, `dim_date`, and `fact_review_sentiment`.
5. **Feature engineering** - A cleaned `app_version_major` field is created from the raw version string. The Play Store API returns whichever version was installed on each reviewer's device, not the version live on the Store at the time — so the data included many old versions from users who hadn't updated, each with very few reviews. Versions were simplified to their major number, and low-volume ones grouped into an "Older/Minor Versions" bucket, so the analysis reflects the app's actual recent releases rather than outdated installs.
5. **Analytical SQL layer** — Four views built on top of the warehouse compute severity-weighted complaint scores, sentiment/rating mismatch detection, rolling sentiment trends, and monthly aspect rankings, using CTEs and window functions (`RANK() OVER PARTITION BY`, `FILTER`, `ROWS BETWEEN`).
6. **Visualization** — Power BI Desktop (Import mode) connects to the warehouse and views, modeling relationships and presenting a 3-page dashboard: Executive Overview, Aspect Deep Dive, and Release Impact.

## Tech stack

| Layer | Tools |
|---|---|
| Scraping & orchestration | Python, `google-play-scraper` |
| NLP | Hugging Face Transformers (zero-shot classification, sentiment analysis) |
| Data warehouse | PostgreSQL (Supabase) |
| Analytics layer | SQL (CTEs, window functions) |
| Visualization | Power BI Desktop (Power Query, DAX, drill-through, custom tooltips) |
| Version control | Git, GitHub |

## Repository structure

```
├── docs/
│   └── pipeline_diagram.svg          # Architecture diagram (embedded above)
├── sql/
│   └── create_star_schema.sql        # DDL for the star schema tables
├── src/
│   ├── ingestion/
│   │   ├── playstore_scraper.py      # Scrapes reviews from the Play Store
│   │   ├── db_loader.py
│   │   └── export_reviews.py
│   ├── nlp_engine/
│   │   ├── sentiment_model.py        # Aspect classification + sentiment scoring
│   │   ├── batch_scorer.py
│   │   └── cleaner.py
│   ├── warehousing/
│   │   └── build_star_schema.py      # Main ETL: scrape → NLP → star schema
│   └── run_pipeline.py               # Incremental daily pipeline runner
├── analytics/
│   ├── data_samples/                 # Sample CSVs at each pipeline stage
│   ├── sql_views/                    # The 4 analytical SQL views, as files
│   └── dashboard/
│       └── Gpay_Sentiment_Dashboard.pbix
├── .gitignore
├── requirements.txt
└── README.md
```

## Dashboard preview

The dashboard is built as four connected pages, opening on a landing page that links out to the three analytical pages, each of which can navigate back home.

### Landing Page
An entry point introducing the project and providing navigation cards to each of the three report pages.

### Page 1 — Executive Overview
A high-level snapshot of the full dataset: KPI cards, five slicers, a 7-day rolling sentiment trend, an aspect-breakdown donut with a custom per-aspect severity tooltip, a sentiment-by-aspect comparison, and a Key Insights panel. Surfaces the standout finding that Account Security & Fraud Risk is only ~1.3% of reviews but ~97% high-severity.

### Page 2 — Aspect Deep Dive
Investigates a single aspect in depth: a severity ranking (RANKX over ALL()), a scatter plot of user rating vs. model sentiment surfacing mismatches, a mismatch-type breakdown, and a table of the actual flagged reviews. Uses FILTER+RELATED and SWITCH(TRUE()) measures, and is reachable via drill-through from Page 1.

### Page 3 — Release Impact
Tracks sentiment across app versions: a KPI row (versions tracked, best/worst version, trend direction), a sentiment-trend-by-version line chart, a version comparison table ranked by sentiment, a "Most-Upvoted Complaints" table (weighting complaints by user upvotes to distinguish widespread from isolated problems), and a Quarter → Month → Week drill-down.

The `.pbix` file is available in `analytics/dashboard/`.

## Future scope

- **Multilingual sentiment support** — The current sentiment model (distilbert-base-uncased-finetuned-sst-2-english) is English-only, and reviews are scraped with an English-language filter at ingestion. Hindi-script reviews are excluded before reaching the pipeline; Hinglish (Hindi in Latin script) may partially pass through and receive unreliable scores. A natural next step is a multilingual model such as XLM-RoBERTa..
- **Longer historical range** — the current dataset spans ~4-5 months due to how far back Play Store scraping can practically reach for a high-volume app. Running the scraper on a recurring schedule (e.g. weekly) would build up a multi-year archive over time, enabling true year-over-year trend analysis.
- **Scheduled refresh** — the dashboard currently runs in Import mode with manual refresh. Publishing to Power BI Service with a scheduled refresh (or a cloud-hosted trigger for the Python pipeline) would make the dashboard update automatically as new reviews come in.
