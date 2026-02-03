# Property Finder Hybrid Recommendation Engine

A high-performance, stateless recommendation engine designed to deliver personalized property listings at scale. The system uses a hybrid scoring strategy that combines **XGBoost-based Learning to Rank (LTR)** with **geospatial waterfall logic** and **real-time personalization**.

**Version:** 2.0.0

---

## Overview

The engine serves ranked property recommendations through a FastAPI service. It is optimized for low latency, horizontal scalability, and daily data refreshes. Recommendations are generated using a multi-stage filtering and scoring pipeline to ensure both relevance and coverage.

---

## System Architecture

The recommendation flow consists of the following stages:

### 1. Strict Filtering (Database Level)
Ensures only valid listings are considered:
- Offering type (Buy / Rent)
- Property type
- Bedroom count
- Price range

This minimizes unnecessary computation and guarantees baseline relevance.

### 2. Geospatial Waterfall
If insufficient results are found in the exact target location, the engine progressively expands the search radius:
- 1.5 km
- 3.5 km
- Up to 10 km

This guarantees that users always receive results, even in low-density areas.

### 3. AI Ranking (60% Weight)
Powered by an **XGBoost Learning to Rank model** (`light_brain.pkl`) that predicts engagement probability using:
- Historical popularity signals
- Listing attributes
- Freshness-adjusted features

### 4. Personalization (Centroid Logic)
Listings are re-ranked in real time using user interaction centroids:
- Average price of interacted listings
- Average latitude and longitude

This provides a tailored experience without requiring stateful storage.

---

## Repository Structure

| File | Role | Description |
|-----|------|-------------|
| `app_2.py` | API Service | FastAPI application serving `/api/v1/recommend` and `/health` endpoints |
| `etl_script.py` | Data Pipeline | Refreshes property inventory and engagement statistics |
| `light_brain.pkl` | AI Model | Trained XGBoost LTR model and feature set |
| `listings.parquet` | Inventory DB | High-speed, compressed store of active listings |
| `Dockerfile` | Containerization | Production-ready Docker image blueprint |
| `requirements.txt` | Dependencies | Pinned Python libraries (FastAPI, Pandas, XGBoost, etc.) |

---

## Installation and Setup

### Local Development

#### Install Dependencies
```bash
pip install -r requirements.txt


Run the API
uvicorn app_2:app --reload --host 0.0.0.0 --port 8000


Containerization

docker build -t property-ai-engine .
docker run -p 8000:8000 property-ai-engine


## Data Lifecycle (ETL)

To maintain recommendation accuracy and freshness, the inventory must be refreshed daily using `etl_script.py`.

### Inventory Extraction
- Fetches listings marked as `online`
- Limited to the last 6 months
- Joins valid pricing and geospatial tower data

### Engagement Weighting

Popularity is computed using weighted Snowplow events:

| Event Type | Weight |
|-----------|--------|
| `content_view` | 1 |
| `lead_click`, `new_projects_lead_click` | 10 |
| `lead_send`, `instapage_lead` | 50 |

### Freshness Boost
A time-based decay function ensures newly published listings receive increased visibility without overwhelming historical performance.

---

## API Usage

### Recommendation Endpoint
**POST** `/api/v1/recommend`

#### Sample Request
```json
{
  "location_name": "Dubai Marina",
  "bedrooms": "2",
  "offering_type": "sale",
  "user_profile": {
    "target_price": 1200000,
    "target_lat": 25.07,
    "target_lon": 55.14,
    "persona": "guest"
  }
}


## Deployment Guidelines

### Platform
- AWS App Runner
- AWS ECS Fargate

### Scalability
- Stateless architecture
- Horizontal auto-scaling based on CPU and memory utilization

### Resource Requirements
- Minimum: **1 vCPU**
- Minimum: **2 GB RAM**
- Required to support Pandas and XGBoost in-memory operations

### CI/CD
- Build and push Docker images to AWS ECR on every merge to the `main` branch

### Automation
- Schedule `etl_script.py` to run nightly
- Refresh `listings.parquet`
- Trigger redeployment or S3 sync to maintain data consistency
