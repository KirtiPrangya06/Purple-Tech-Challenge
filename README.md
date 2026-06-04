# Store Intelligence — Retail Analytics from CCTV

Real-time store analytics pipeline: CCTV footage → detection → events → API → dashboard.

**North Star Metric:** Offline Store Conversion Rate = Customers who purchased ÷ Total unique visitors

---

## Quick Start (5 commands)

```bash
# 1. Clone the repo
git clone <repo-url> store-intelligence && cd store-intelligence

# 2. Place dataset files
mkdir -p data/clips
# Copy .mp4 clips → data/clips/
# Copy store_layout.json → data/store_layout.json
# Copy pos_transactions.csv → data/pos_transactions.csv

# 3. Start the API
docker compose up -d

# 4. Run the detection pipeline against the clips
./pipeline/run.sh data/clips data/store_layout.json data/events.jsonl http://localhost:8000

# 5. Open the live dashboard
docker compose logs -f dashboard
# OR run locally: python3 dashboard/live_dashboard.py --store STORE_BLR_002
```

API docs available at: **http://localhost:8000/docs**

---

## Prerequisites

- Docker + Docker Compose (v2.x)
- Python 3.10+ (for running the detection pipeline locally)
- ~4GB disk space (YOLOv8s model download ~22MB, video clips ~several GB)

The detection pipeline requires additional Python packages installed automatically by `run.sh`:
```
ultralytics   opencv-python-headless   numpy   requests
```
Optional (for better Re-ID accuracy):
```
torchreid   torch
```

---

## Full Setup Guide

### Step 1 — Clone and enter the project

```bash
git clone <repo-url> store-intelligence
cd store-intelligence
```

### Step 2 — Place the dataset

```bash
mkdir -p data/clips

# Expected layout:
# data/
#   clips/
#     STORE_BLR_002__CAM_ENTRY_01__2026-03-03T09-00-00Z.mp4
#     STORE_BLR_002__CAM_FLOOR_01__2026-03-03T09-00-00Z.mp4
#     STORE_BLR_002__CAM_BILLING_01__2026-03-03T09-00-00Z.mp4
#     ... (5 stores × 3 cameras = 15 clips)
#   store_layout.json
#   pos_transactions.csv
```

### Step 3 — Start the API

```bash
docker compose up -d

# Verify it's healthy:
curl http://localhost:8000/health
```

Expected response:
```json
{"status": "healthy", "database": "ok", "stores": [], "uptime_seconds": 5.1}
```

### Step 4 — Run the detection pipeline

```bash
# Process all clips → emit events → POST to API in real time
./pipeline/run.sh data/clips data/store_layout.json data/events.jsonl http://localhost:8000
```

The pipeline will:
1. Load YOLOv8s (downloads ~22MB on first run)
2. Process each clip at 5fps effective rate
3. Emit JSONL events to `data/events.jsonl`
4. POST events to `http://localhost:8000/events/ingest` in batches of 100

Progress is logged to stdout. Typical processing time: ~2–5 minutes per 20-minute clip on CPU.

**To process a single clip:**
```bash
python3 pipeline/detect.py data/clips data/store_layout.json data/events.jsonl \
  --clip data/clips/STORE_BLR_002__CAM_ENTRY_01__2026-03-03T09-00-00Z.mp4 \
  --api-url http://localhost:8000
```

**To ingest a pre-existing events.jsonl without re-running detection:**
```bash
python3 pipeline/ingest_events.py data/events.jsonl http://localhost:8000
```

**To simulate real-time event streaming:**
```bash
python3 pipeline/ingest_events.py data/events.jsonl http://localhost:8000 \
  --simulate-realtime --batch-size 50
```

### Step 5 — Query the API

```bash
# Store metrics (today)
curl http://localhost:8000/stores/STORE_BLR_002/metrics | python3 -m json.tool

# Conversion funnel
curl http://localhost:8000/stores/STORE_BLR_002/funnel | python3 -m json.tool

# Zone heatmap
curl http://localhost:8000/stores/STORE_BLR_002/heatmap | python3 -m json.tool

# Active anomalies
curl http://localhost:8000/stores/STORE_BLR_002/anomalies | python3 -m json.tool

# Service health
curl http://localhost:8000/health | python3 -m json.tool
```

---

## Live Dashboard (Part E)

The dashboard shows real-time metrics updating as events flow in.

**Terminal dashboard (included):**
```bash
# Via Docker (starts automatically with docker compose up)
docker compose logs -f dashboard

# Or run locally
pip install rich requests
python3 dashboard/live_dashboard.py --api http://localhost:8000 --store STORE_BLR_002 --refresh 5
```

The dashboard shows:
- 📊 Live metrics: visitors, conversion rate, queue depth, abandonment rate
- 🔽 Conversion funnel with drop-off percentages and ASCII bars
- 🔥 Zone heatmap (top 8 zones by engagement score)
- 🚨 Active anomalies colour-coded by severity (INFO/WARN/CRITICAL)

---

## Running Tests

```bash
# Install test dependencies
pip install pytest pytest-asyncio pytest-cov httpx

# Run all tests with coverage
pytest tests/ -v --cov=app --cov-report=term-missing

# Run a specific test file
pytest tests/test_pipeline.py -v
pytest tests/test_metrics.py -v
pytest tests/test_anomalies.py -v
```

Expected coverage: >70% statement coverage across `app/`.

---

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/events/ingest` | POST | Ingest batch of events (max 500). Idempotent by `event_id`. |
| `/stores/{id}/metrics` | GET | Unique visitors, conversion rate, dwell, queue, abandonment |
| `/stores/{id}/funnel` | GET | Entry → Zone → Billing → Purchase with drop-off % |
| `/stores/{id}/heatmap` | GET | Zone frequency + dwell, normalised 0–100 |
| `/stores/{id}/anomalies` | GET | Active anomalies with severity + suggested actions |
| `/health` | GET | DB status, feed lag per store, uptime |
| `/docs` | GET | Auto-generated OpenAPI UI (Swagger) |

All endpoints accept `?window_hours=N` (1–168) to control the lookback window. Default: 24 hours.

---

## Project Structure

```
store-intelligence/
├── pipeline/
│   ├── detect.py          # Main detection + tracking script
│   ├── tracker.py         # ByteTrack-style tracking + Re-ID
│   ├── emit.py            # Event schema + emission
│   ├── staff_detector.py  # Torso colour histogram staff detection
│   ├── zone_classifier.py # Homography + polygon zone mapping
│   ├── ingest_events.py   # Manual batch ingest script
│   └── run.sh             # One-command pipeline runner
├── app/
│   ├── main.py            # FastAPI entrypoint + middleware
│   ├── models.py          # Pydantic schemas
│   ├── db/database.py     # SQLite schema + connection management
│   ├── routers/
│   │   ├── events.py      # POST /events/ingest
│   │   ├── stores.py      # GET /stores/{id}/*
│   │   └── health.py      # GET /health
│   └── services/
│       ├── ingestion.py   # Dedup + store events
│       ├── metrics.py     # Real-time metric computation
│       ├── funnel.py      # Funnel + session dedup
│       ├── heatmap.py     # Zone heatmap
│       ├── anomalies.py   # Anomaly detection
│       └── health.py      # Service health check
├── tests/
│   ├── test_pipeline.py   # Detection + emission tests
│   ├── test_metrics.py    # API endpoint tests
│   └── test_anomalies.py  # Anomaly detection tests
├── dashboard/
│   └── live_dashboard.py  # Rich terminal dashboard
├── docs/
│   ├── DESIGN.md          # Architecture + AI-assisted decisions
│   └── CHOICES.md         # 3 key decisions with full reasoning
├── data/                  # Dataset (not committed to repo)
│   ├── clips/             # .mp4 CCTV clips
│   ├── store_layout.json
│   ├── pos_transactions.csv
│   └── events.jsonl       # Generated by pipeline
├── docker-compose.yml
├── Dockerfile
├── Dockerfile.dashboard
├── requirements.txt
└── README.md
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `./data/store_intelligence.db` | SQLite file path |
| `PORT` | `8000` | API listen port |
| `LOG_LEVEL` | `INFO` | Logging level |
| `API_URL` | `http://localhost:8000` | API URL (used by dashboard) |
| `STORE_ID` | `STORE_BLR_002` | Default store for dashboard |

---

## Troubleshooting

**`docker compose up` fails with port conflict:**
```bash
# Change port in docker-compose.yml: "8080:8000" instead of "8000:8000"
```

**Detection pipeline can't find YOLOv8:**
```bash
pip install ultralytics
# Model downloads automatically on first run (~22MB)
```

**API returns 503 on metrics:**
```bash
# Check DB file exists and is writable
ls -la data/store_intelligence.db
docker compose logs api
```

**No events in API after running pipeline:**
```bash
# Verify events.jsonl was created
wc -l data/events.jsonl

# Manually ingest
python3 pipeline/ingest_events.py data/events.jsonl http://localhost:8000
```

---

## Architecture Decisions

See [`docs/DESIGN.md`](docs/DESIGN.md) for the full architecture overview including AI-assisted decisions.

See [`docs/CHOICES.md`](docs/CHOICES.md) for the three key decisions: model selection, event schema design, and API architecture.
