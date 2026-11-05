# Zomato Analytics & AI

An end-to-end analytics project that combines Snowflake, dbt, Apache Airflow, and OpenAI to transform Zomato-style order and review data into business marts and AI-assisted insights.

## What this project does

- Loads restaurant, customer, food, menu, order, order-item, and review data into Snowflake raw tables.
- Uses dbt to build staging views, customer/restaurant/food/date dimensions, order fact tables, and reporting marts.
- Runs a scheduled Airflow DAG to reload raw data, build core models, enrich reviews, and build AI models.
- Uses OpenAI to classify review sentiment, topics, and key issues, then summarizes them in a dbt mart.
- Includes two Streamlit apps: semantic search and Q&A over reviews, and natural-language questions translated into Snowflake SQL.

## Architecture

```text
CSV datasets / Snowflake stage
             |
             v
       Snowflake RAW
             |
             v
    dbt staging views
             |
             v
 dbt dimensions and facts -------> dbt reporting marts
             |
             +<---- Airflow orchestrates the daily pipeline
             |
             +----> OpenAI review enrichment
                        |
                        v
                   Snowflake AI
                        |
                        v
                 dbt review insights

Snowflake marts/reviews ---> Streamlit Q&A apps
```

Airflow's `zomato_batch` DAG is scheduled daily. It reloads raw tables from the configured Snowflake stage, builds non-AI dbt models, enriches a sample of reviews, and builds AI-tagged dbt models.

## Repository layout

| Path | Purpose |
| --- | --- |
| `ai/enrich_reviews.py` | Classifies review sentiment, topic, and key issue, then saves results to `ZOMATO.AI.REVIEW_ENRICHED`. |
| `ai/rag_chat.py` | Streamlit review search and Q&A using OpenAI embeddings and Snowflake review data. |
| `ai/text_to_sql.py` | Streamlit natural-language-to-SQL demo for Snowflake analytics tables. |
| `airflow/dags/zomato_batch.py` | Daily orchestration for raw reload, dbt builds, and review enrichment. |
| `airflow/Dockerfile` | Airflow image with the Snowflake provider, OpenAI, and an isolated dbt environment. |
| `airflow/docker-compose.yaml` | Local Airflow 3.x and Postgres services. |
| `data/` | Instructions for supplying local CSV inputs; source datasets are not stored in Git. |
| `zomato/models/staging/` | dbt views that standardize raw source tables. |
| `zomato/models/marts/` | dbt dimensions, facts, and analytics marts. |
| `zomato/profiles.example.yml` | Safe dbt profile template; copy it to the ignored local `zomato/profiles.yml`. |

## Technology

- **Warehouse:** Snowflake
- **Transformations:** dbt Core with `dbt-snowflake`
- **Orchestration:** Apache Airflow 3.x, Docker Compose, and Postgres
- **AI:** OpenAI chat completions and embeddings
- **Applications:** Streamlit, pandas, NumPy, and PyArrow

## Prerequisites

- Docker Desktop with Docker Compose
- A Snowflake account, warehouse, role, database, and credentials
- An OpenAI API key for review enrichment and the AI apps
- Python 3.10 or later for running the Streamlit apps outside Docker
- Access to the Snowflake raw tables and stage expected by the DAG (see [Snowflake setup](#snowflake-setup))

## Configuration and secrets

Do not commit credentials. `.env` files and the local `zomato/profiles.yml` are ignored by Git. Use the checked-in templates:

1. Copy `airflow/.env.example` to `airflow/.env` and fill in your local credentials.
2. Copy `zomato/profiles.example.yml` to `zomato/profiles.yml`.
3. Set the environment variables referenced by the dbt profile in your shell or local environment:
   - `SNOWFLAKE_ACCOUNT`
   - `SNOWFLAKE_USER`
   - `SNOWFLAKE_PASSWORD`
   - `SNOWFLAKE_WAREHOUSE` (defaults to `ZOMATO_WH` in the template)
   - `SNOWFLAKE_DATABASE` (defaults to `ZOMATO`)
   - `SNOWFLAKE_SCHEMA` (defaults to `STAGING`)
   - `SNOWFLAKE_ROLE` (defaults to `DBT_ROLE`)
   - `OPENAI_API_KEY` for AI features
   - `SAMPLE_N` (optional; Airflow review-enrichment sample size, defaults to `5`)

For running a Streamlit app locally, make the same variables available in the process environment or in a `.env` file in the project root. Never commit your filled-in environment file or profile.

## Snowflake setup

The project expects Snowflake database `ZOMATO` and raw tables in schema `RAW`: `RESTAURANTS`, `USERS`, `FOOD`, `MENU`, `ORDERS`, `ORDER_ITEMS`, and `REVIEWS`. The Airflow DAG also expects the `ZOMATO.RAW.ZOMATO_RAW_STAGE` stage with matching folders for each entity.

Configure the required Snowflake database objects, file formats, stage, permissions, warehouse, role, and credentials in your own Snowflake account before running the pipeline. Those account-specific provisioning scripts are not included here. The large source CSV datasets are intentionally excluded from this repository. Obtain the data separately, place it in the layout described in [`data/README.md`](data/README.md), and upload it to your stage using your organization's approved process.

The AI models additionally expect `ZOMATO.AI.REVIEW_ENRICHED`. The enrichment script creates this schema and table if needed. dbt reads the resulting table as the `ai.review_enriched` source.

## Run the dbt project locally

From the repository root, after configuring the profile and Snowflake environment variables:

```bash
cd zomato
dbt debug --profiles-dir .
dbt build --profiles-dir .
```

`dbt build` runs models and configured data tests. The project materializes staging models as views and marts as tables. The `target/`, `dbt_packages/`, and `logs/` directories are generated locally and are not intended for version control.

## Run the Airflow pipeline

From the `airflow/` directory, after creating `airflow/.env` and configuring `zomato/profiles.yml`:

```bash
docker compose build
docker compose up -d
```

Open [http://localhost:8080](http://localhost:8080), sign in with the local account configured in `docker-compose.yaml`, and enable the `zomato_batch` DAG. The compose setup initializes a local Airflow account with the sample `admin` / `admin` credentials; change these before exposing Airflow beyond your machine.

Useful commands, run from `airflow/`:

```bash
docker compose ps
docker compose logs -f scheduler
docker compose down
```

The compose stack stores Airflow metadata in a local Postgres volume. `docker compose down` stops the services without deleting that volume.

## Run the Streamlit apps locally

Install the app dependencies in a virtual environment:

```bash
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# macOS/Linux:
# source .venv/bin/activate
python -m pip install openai snowflake-connector-python python-dotenv numpy pandas pyarrow streamlit
```

From the repository root, with the required environment variables available:

```bash
streamlit run ai/rag_chat.py
```

In another terminal, launch the natural-language SQL app with:

```bash
streamlit run ai/text_to_sql.py
```

The review app fetches reviews from Snowflake on first run and caches embeddings in `ai/review_embeddings.parquet`. That cache is generated data and is excluded from Git.

The SQL app is a demonstration, not a production security boundary. Its generated-query checks are intentionally simple; do not expose it to untrusted users or connect it to a privileged production role.

## Data models

### Staging

`stg_restaurants`, `stg_users`, `stg_food`, `stg_menu`, `stg_orders`, `stg_order_items`, and `stg_reviews` normalize the corresponding raw Snowflake sources.

### Dimensions and facts

- Dimensions: `dim_customer`, `dim_date`, `dim_food`, and `dim_restaurants`
- Facts: `fct_orders` and `fct_order_items`

### Marts

- `mart_daily_city_revenue`: daily city-level order counts, delivery counts, cancellation rate, GMV, and average order value.
- `mart_delivery_sla`: delivered-order counts and delivery-time percentiles by city and order hour.
- `mart_restaurant_performance`: restaurant-level order volume, delivered revenue, ratings, and delivery time.
- `mart_review_insights`: review counts and average sentiment/star ratings by city, topic, and sentiment, including flagged issues.

## Tests and generated files

Run dbt model tests as part of the build:

```bash
cd zomato
dbt test --profiles-dir .
```

Current schema tests include uniqueness, non-null, accepted-value, and relationship checks for selected staging and mart models. Airflow logs, dbt artifacts, Python bytecode, local environment files, and generated review embeddings are excluded from Git.

## Notes

- Snowflake object names, role, warehouse, and the file stage are configured for this project and may need adaptation for another account.
- The default Airflow compose credentials and JWT secret are for local development only. Replace them before any non-local deployment.
- AI features incur OpenAI API usage and Snowflake compute costs.
- The analytics and AI outputs depend on the quality and permissions of the source data in your Snowflake account.
