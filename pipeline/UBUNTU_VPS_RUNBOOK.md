# Ubuntu VPS runbook

[← Documentation home](README.md) · [Operator guide](../airflow_dags/README.md)

Deploy and run the Sites Intelligence pipeline on **Ubuntu Linux** (or WSL). Airflow is not supported on native Windows.

---

## Prerequisites

- Ubuntu 22.04+ (or WSL2)
- Python 3.11+ with project venv
- Repo cloned to e.g. `~/sites-intelligence-pipeline`
- `.env` copied from `.env.example` and filled in (GCS, BigQuery, Postgres, Neo4j, OpenRouter)

---

## One-time Airflow setup

### 1. Install dependencies

```bash
cd ~/sites-intelligence-pipeline
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt   # or your project's install command
```

### 2. Configure `.env`

Set at minimum:

- `AIRFLOW_HOME=/home/YOUR_USER/airflow`
- `GOOGLE_APPLICATION_CREDENTIALS`, `GCS_BUCKET`, `BQ_PROJECT`, `BQ_DATASET`
- `DB_*` for pipeline Postgres (DAG 07–09)
- `OPENROUTER_API_KEY` (DAGs 02 sponsor AI, 05, 08)
- `NEO4J_*` (DAG 10)

`AIRFLOW__CORE__DAGS_FOLDER` and `AIRFLOW__CORE__PLUGINS_FOLDER` can be omitted — `bootstrap_airflow_env.py` sets them from the repo root.

Optional: `AIRFLOW_DB_NAME` for Postgres-backed Airflow metadata (otherwise SQLite in `AIRFLOW_HOME`).

### 3. Symlink local settings

```bash
ln -sf ~/sites-intelligence-pipeline/airflow_local_settings.py ~/airflow/airflow_local_settings.py
```

`airflow_local_settings.py` calls `apply_airflow_environment()` so paths and the metadata DB come from `.env` without exporting variables manually.

### 4. Initialize Airflow (first time)

```bash
cd ~/sites-intelligence-pipeline
set -a && source .env && set +a
airflow db migrate
```

### 5. Start scheduler and webserver

```bash
set -a && source .env && set +a
airflow webserver -p 8080 &
airflow scheduler
```

Use systemd or supervisor in production instead of background jobs.

---

## Running the pipeline

1. Trigger **`dag_01_bronze_layer`** (or wait for the weekly schedule).
2. DAGs 02–10 chain automatically on success.

Monitor the Airflow UI at `http://YOUR_VPS:8080`.

---

## Local task debugging (no scheduler)

From the repo root, run individual tasks with the same operators and `.env`:

```bash
python run_local.py --list-dags
python run_local.py --dag dag_01 --list-tasks
python run_local.py --dag dag_01 --task stage_ror_to_gcs --with-upstream
```

See [`run_local.py`](../run_local.py) for flags (`--param`, `--dry-run`, etc.).

---

## Troubleshooting

| Issue | Check |
|-------|--------|
| Import errors in DAGs | `PYTHONPATH` includes repo root; symlink `airflow_local_settings.py` |
| Postgres read-only in DAG 07/08 | Use direct writer port; set `DB_WRITE_HOST`; try `DB_VERIFY_WRITABLE=true` |
| OpenRouter failures | `OPENROUTER_API_KEY` set; rate limits on first full populate |
| OpenAlex bronze timeout | Large S3 snapshot — allow many hours on first DAG 01 run |

---

## Related

- [Operator / developer guide](../airflow_dags/README.md)
- [Pipeline notes](pipeline-notes.md)
- [Pipeline overview](pipeline-overview.md)

[← Back to documentation home](README.md)
