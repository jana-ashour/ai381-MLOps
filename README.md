



## Run it
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
python -m src.train.train        # trains the model into models/
pytest -q                        # all tests should pass
uvicorn src.app.main:app --port 8000
curl -s localhost:8000/health
docker build -t churn-api:local . && docker run --rm -p 8000:8000 churn-api:local
```


- MLflow in Docker Compose; log params, metrics, a data hash, the signature and the model
- Three candidate models compared per slice, latency and size; `MODEL_SELECTION.md`
- The `@champion` alias and a training workflow with a promotion gate
- Deploying the champion image: to Azure Container Apps with OIDC (optional), or on your own laptop with Docker Compose, with a smoke test either way
- Lineage: image tag → commit SHA → MLflow run ID

## Files that help you (read their comments first)
| File | What it is |
|---|---|
| `docker-compose.mlflow.yml` | The MLflow server. `docker compose -f docker-compose.mlflow.yml up -d --wait`, then open http://localhost:5000 |
| `src/train/train_mlflow.py` | Trains the candidates and logs every run. Your work is marked `TODO` |
| `scripts/promote.py` | The promotion gate. Exit code 1 means a model was blocked. Your thresholds are marked `TODO` |
| `.github/workflows/train.yml` | Runs the training and the gate on every pull request |
| `docker-compose.deploy.yml` and `scripts/smoke_test.sh` | The local deployment and the smoke test (works for Azure too) |
| `.github/workflows/deploy-azure.yml.example` | The Azure option, switched off until you rename it |
| `MODEL_SELECTION.md` | The template for your decision |


