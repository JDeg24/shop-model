# Shop model

Predicts whether an item in the shop will sell, from its price.

## Requirements

Python 3.12 with the venv module. On Ubuntu:

```bash
sudo apt install python3 python3-venv
```

## Install

Run these from this folder, the one with pyproject.toml:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
python3 -m pip install -e ".[dev]"
cp env.example .env
```

## Test

```bash
pytest
```

## Train

```bash
shop-train
```

The model is saved to models/model.joblib. Shared settings are in
config/config.yaml; settings for your machine go in .env.


## Monitoring

The monitoring services (Prometheus and Grafana) are in the monitoring
folder and start together with the API and the database:

```bash
docker compose up -d --build
docker compose ps
```

Four containers should be running: api, db, prometheus and grafana.

- Dashboard (Grafana): http://localhost:3000, opens on the Model service
  dashboard, no login needed
- Prometheus: http://localhost:9090 (Status, then Target health, should
  show model-api as UP)
- The API's own metrics: http://localhost:8888/metrics

In a VirtualBox VM, ports 3000 and 9090 need port-forwarding rules, as
for 8888.

## Generating traffic

With the virtual environment active, from this folder:

```bash
python3 monitoring/traffic.py --field price --mean 23 --sd 12 --duration 300
```

To simulate drift (prices rising), run it again with a higher mean:

```bash
python3 monitoring/traffic.py --field price --mean 35 --sd 12 --duration 300
```

Stop it early with Ctrl+C. `python3 monitoring/traffic.py --help` lists
all options.

## Stopping

```bash
docker compose down
```

This keeps the stored predictions and metrics, which live in volumes.
`docker compose down -v` deletes them as well.

## The pipeline (Session 5, Stage 12)

Every check runs with one command, in order, stopping at the first failure:

```
make install    # pinned requirements, then the package with its dev tools
cp env.example .env
make ci         # lint, test, validate, gate, build
```

| Target | What it does | Fails when |
|---|---|---|
| `make lint` | ruff checks the code | a style or correctness rule is broken |
| `make test` | pytest, including the property-based range test | any test fails |
| `make validate` | checks `data/prices_v2.csv` and `data/holdout.csv` against the schema | a row is invalid or a column is unexpected |
| `make gate` | trains on rows before the holdout and scores the frozen holdout | exit 1: below the threshold; exit 2: the holdout changed |
| `make build` | builds `shop-model:<version>` and `shop-model:<version>-<commit>`, the version read from `pyproject.toml` | the image does not build |

Compare the gate with the random split of Stage 7: `make gate GATE_SPLIT=random`.

The gate's metric and threshold live in `config/config.yaml` (`gate_metric`, `gate_threshold`), next to the holdout's start date and SHA-256 fingerprint. Never edit `data/holdout.csv`: the gate refuses to run if its fingerprint changes.

Start the whole stack with the version from `pyproject.toml`: `make up`.

## Continuous integration

GitHub Actions runs `.github/workflows/ci.yml` on every push and pull
request. It sets up Python, installs the project, copies `env.example` to
`.env`, and runs `make ci` to lint, test, validate the data, check the
model quality gate and build the image. The pipeline stops if a step fails.

After the checks pass on a push to `main`, the workflow publishes the image
to GitHub Container Registry at `ghcr.io/<owner>/shop-model`, where the
owner name is lowercase. It pushes two tags: `<version>` and
`<version>-<commit>`. The workflow uses GitHub's `GITHUB_TOKEN` to log in.

To release a new version, change the project version in `pyproject.toml`
and push the change to `main`. A successful workflow publishes the image
with the new version tags.
