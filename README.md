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
