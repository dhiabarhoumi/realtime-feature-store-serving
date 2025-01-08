# Real-Time Feature Store & Online Serving

A production-quality real-time feature store with online serving capabilities, demonstrating streaming feature engineering, training-serving parity, and low-latency inference for recommender/risk models.

## Problem & Architecture

This system produces and serves low-latency features for online recommender/risk models with guaranteed training-serving parity. It ensures point-in-time correctness, feature freshness, and sub-5ms online lookups with end-to-end inference latency under 120ms at 500 RPS.

### What it does (end-to-end)
1. **Ingest**: Realistic clickstream events into Redpanda/Kafka topic `events_raw`
2. **Stream Processing**: Spark Structured Streaming computes sliding/tumbling window aggregations 
3. **Feature Store**: Feast manages offline (Parquet) and online (Redis) feature storage
4. **Training**: Point-in-time feature retrieval with MLflow model tracking
5. **Serving**: FastAPI with Feast middleware for sub-5ms feature lookups
6. **Quality Assurance**: Great Expectations for skew detection and parity testing

```
[Replay/Live] → Kafka → Spark Streaming → Feast → Redis → FastAPI
                            ↓
                       Offline Parquet → Training → MLflow
```

## KPIs & Evidence

- **p95 online read ≤ 5ms** (Redis feature lookups)
- **p95 E2E inference ≤ 120ms** at 500 RPS 
- **Parity error < 0.5%** between offline/online features
- **Evidence**: `artifacts/reports/` contains parity reports, k6 performance results, Grafana dashboards

## Quick Start

### 1. Setup Environment
```bash
make setup
cp .env.example .env
make up  # starts redpanda, redis, prometheus, grafana
```

### 2. Create Topic & Start Data Producer
```bash
rpk topic create events_raw
make produce  # replays data/events.csv.gz at 500 EPS
```

### 3. Run Streaming Pipeline
```bash
make stream  # spark structured streaming job
```

### 4. Setup & Materialize Features
```bash
make feast-apply  # apply feast feature definitions
make feast-mat    # materialize features to online store
```

### 5. Train Model
```bash
make train  # point-in-time feature retrieval → model training → MLflow
```

### 6. Serve Model
```bash
make serve  # FastAPI server on :8080
```

### 7. Test & Validate
```bash
make test    # unit + e2e tests
make parity  # offline/online feature parity validation
make perf    # k6 load test at 500 RPS
```

### All-in-one
```bash
make all  # runs complete pipeline end-to-end
```

## API Usage

```bash
# Health check
curl http://localhost:8080/healthz

# Prediction with feature lookup
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "u_10293",
    "product_id": "sku_12345", 
    "session_id": "s_9c1d..."
  }'

# Response
{
  "score": 0.742,
  "features_freshness_ms": 4300,
  "model_version": "1.0.3"
}
```

## Features

### Entities & TTLs
- **user_id** (TTL: 7 days) - End user behavior patterns
- **product_id** (TTL: 3 days) - Product performance metrics  
- **session_id** (TTL: 2 hours) - Session-level interactions

### Feature Examples
- **User**: `user_views_5m`, `user_add2cart_1h`, `user_cr_24h` (conversion rate)
- **Product**: `prod_ctr_24h` (click-through rate), `prod_rev_24h`
- **Session**: `sess_duration_s`, `sess_unique_products`

All features include event timestamps and watermarking for late data handling (10min tolerance).

## Data Schema

Events are ingested as JSON to `events_raw` topic:

```json
{
  "event_id": "e_8d8c0d8e...",
  "event_ts": "2025-01-15T12:31:22.314Z",
  "user_id": "u_10293",
  "session_id": "s_9c1d...",
  "event_type": "page_view",
  "product_id": "sku_12345",
  "price": 29.9,
  "currency": "EUR",
  "device": {"mobile": false, "os": "Windows"},
  "geo": {"country": "DE", "city": "Munich"}
}
```

## Architecture Details

### Streaming (Spark Structured Streaming)
- Reads from Kafka with exactly-once semantics
- Event-time processing with 10min watermark for late arrivals
- Sliding/hopping windows for aggregations
- Sessionization by user_id + session_id
- Checkpointing for fault tolerance

### Feature Store (Feast)
- **Offline**: Parquet tables partitioned by event_date
- **Online**: Redis for sub-5ms lookups
- Point-in-time joins prevent data leakage
- TTL-based feature expiration

### Serving (FastAPI + Redis)
- Feast middleware fetches features in batch
- Model prediction with XGBoost/LightGBM
- Prometheus metrics for observability
- Graceful fallbacks for missing features

## Monitoring & Observability

- **Prometheus Metrics**: Feature lookup latency, inference latency, cache hit rates
- **Grafana Dashboards**: Real-time performance monitoring
- **Great Expectations**: Data quality validation
- **MLflow**: Model tracking and versioning

## Testing Strategy

- **Unit Tests**: Point-in-time correctness, window aggregations
- **E2E Tests**: Full pipeline from ingestion to serving
- **Performance Tests**: k6 load testing at 500 RPS
- **Parity Tests**: Offline vs online feature consistency

## Limitations

- Local single-broker setup; production requires replication
- Feast push sources optional; uses materialization from offline store
- Single-pod serving; horizontal scaling needs load balancer

## Development

```bash
# Install dependencies
pip install -r requirements.txt

# Run tests
pytest -v

# Pre-commit hooks
pre-commit install
pre-commit run --all-files

# Docker build
docker build -t realtime-fs .
```

## Repository Structure

```
realtime-feature-store-serving/
├── README.md                   # This file
├── project.yaml               # Project metadata
├── Makefile                   # Build automation
├── docker/                    # Container orchestration
├── feastrepo/                 # Feast feature definitions
├── pipelines/                 # Streaming jobs (Spark/Flink)
├── src/                       # Application code
│   ├── serving/              # FastAPI model server
│   ├── training/             # Model training pipeline
│   ├── parity/               # Offline/online validation
│   └── tests/                # Test suites
├── data/                      # Sample datasets
└── artifacts/                 # Generated reports & evidence
```

---

*For production deployment, see [docs/deployment.md](docs/deployment.md)*