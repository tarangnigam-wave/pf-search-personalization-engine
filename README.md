# 🏠 Property Finder Recommendation Engine (v6.6.0)

## 1. Project Overview
This is a production-ready **Hybrid Recommendation Engine** designed to rank real estate listings based on a combination of:

- **User Intent (Filters)**
- **Geospatial Proximity**
- **AI-Driven Popularity**

Unlike standard search engines that simply sort by price, this engine uses a **Learning-to-Rank (LTR) XGBoost Model** to predict which properties are most likely to generate a lead, balancing **Best Match** with **Highest Quality**.

---

## 2. System Architecture

The system runs as a high-performance **FastAPI** service (`app_7.py`) consisting of three layers.

---

### Layer 1: The Data Layer (`listings.parquet`)
- **Storage:** Parquet format for ultra-fast columnar reads (~72k+ listings).
- **Normalization on Startup:**
  - `furnished_flag`: Maps `"yes"`, `"true"`, `"1"` → `1`
  - `completion_status`: Maps `"ready"`, `"completed"` → `"completed"`
  - `listing_date`: Parsed for freshness calculations
- **Geospatial Index:**
  - Pre-calculated centroids (Lat/Lon) for known locations (e.g., *Dubai Marina*)
  - Enables instant radius-based searches

---

### Layer 2: The Logic Layer (Filters & Search)

Before AI scoring, strict business rules narrow down the dataset:

1. **Hard Filters**
   - Category (Buy / Rent)
   - Price Range
   - Bedrooms
   - Bathrooms
   - Area

2. **Geospatial Logic**
   - **Radius Search:** Uses Haversine Distance between user and listings
   - **Keyword Match:**  
     Example: searching `"Marina"` resolves to the Dubai Marina centroid and applies a **1.5km – 3.5km radius**

3. **Trust Logic**
   - Filters for **Super Agents**
   - Based on a dynamic `trust_score` threshold (Top 5% of agents)

---

### Layer 3: The AI Brain (`light_brain.pkl`)

This is the **core intelligence** of the system.

- **Model:** XGBoost Classifier
- **Objective:** Predict probability of conversion (`lead_submission_count > 0`)

#### Key Features Used
- `smart_popularity_score` – weighted aggregate of views, clicks, and leads
- `quality_score` – internal listing quality rating
- `trust_score` – agent performance metric
- `freshness_score` – decay-based boost for newer listings

#### Categorical Features
- `offering_type`
- `property_type`
- `location_id`
- `furnished_flag`

---

## 3. Scoring & Sorting Logic

The engine supports two ranking modes.

---

### Mode A: Featured (Smart AI Rank)

Default behavior when no explicit sort is selected.

\[
\text{Final Rank} =
\text{AI Probability Score}
- (\text{Distance in KM} \times 0.1)
+ \text{Luxury Boost}
\]

**Components**
- **AI Probability:** Likelihood (0–100%) that a user will inquire
- **Distance Penalty:** Slight penalty for farther listings
- **Luxury Boost:** Small visibility boost for properties > 5,000,000 AED

---

### Mode B: Explicit Sort

When a user selects a manual sort, AI ranking is bypassed for ordering, but still returned as metadata.

Supported sorts:
- **Price:** Low → High / High → Low
- **Bedrooms:** Least → Most
- **Newest:** By `listing_date`

---

## 4. API Interface

**Endpoint**


POST /api/v1/search


The API accepts a strict JSON structure derived from a Protocol Buffers contract.

---

## Sample Payload
```json
{
  "filters": {
    "category_id": 2,
    "min_price": 50000,
    "max_price": 150000,
    "keywords": ["Marina"],
    "number_of_bedrooms": [1, 2],
    "is_super_agent": false
  },
  "pagination": {
    "page": 1,
    "limit": 20
  },
  "sorting": {
    "sort": "featured"
  }
}


5. Setup & Requirements
Dependencies

fastapi
uvicorn
pandas
xgboost
joblib
pyarrow


Required Files
listings.parquet – Listings database
light_brain.pkl – Trained model (must match feature columns in app_7.py)
uvicorn app_7:app --host 0.0.0.0 --port 8000 --reload



