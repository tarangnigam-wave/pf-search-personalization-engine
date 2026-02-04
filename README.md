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


## Part 3: Future Scope (Phase 2)

---

## A. Rental Period Logic (`price_type`)

### Current
- Listings are filtered only by `category_id` (Buy vs Rent).
- Rental duration (Daily / Weekly / Yearly) is not distinguished.

### Problems
- Daily rentals (e.g., **500 AED/day**) can appear alongside yearly rentals (e.g., **50,000 AED/year**).
- This leads to misleading price comparisons and a poor user experience.

### Future
- Introduce explicit `price_type` handling:
  - `daily`
  - `weekly`
  - `yearly`
- Enforce filtering and validation so users only see listings within the same rental period.
- Align frontend filters, backend logic, and search behavior.

> **Note:** The current AI model (`light_brain.pkl`) does **not** use `price_type` as a feature.  
> This enhancement will initially be implemented at the **logic layer**, with model retraining planned in a later phase.

---

## B. Hierarchical Location Search

### Current
- Location matching is handled via:
  - Radius-based geospatial search, or
  - Simple keyword matching (e.g., `"Marina"`).
- No understanding of parent–child location relationships.

### Limitations
- Searching for a parent location (e.g., **Dubai**) does not automatically include sub-locations.
- Keyword matching is brittle and can be ambiguous.

### Future
- Introduce a `full_location_path` field (e.g.):

# Property Finder AI Recommendation Engine (v6.6.0)

> **Status:** Production Ready (Phase 1)  
> **Type:** Hybrid Filtering + Learning-to-Rank (LTR) AI  
> **Tech Stack:** Python, FastAPI, XGBoost, Pandas, Parquet

---

## 1. Executive Summary

This project represents a shift from traditional "Database Search" (SQL WHERE clauses) to an **AI-Driven Discovery Engine**.

Instead of simply returning listings that match a filter, this engine calculates the **Probability of Conversion** for every listing. It asks: *"Among the 500 apartments matching the user's criteria, which ones are most likely to result in a Lead?"*

It achieves this through a **3-Layer Architecture**:
1.  **Strict Filtering:** Hard constraints (Price, Beds, Location).
2.  **Geospatial Intelligence:** Radius-based search with text fallbacks.
3.  **AI Re-Ranking:** XGBoost model predicting user interest based on behavioral signals.

---

## 2. System Architecture

### A. The Data Pipeline (`pipeline.py`)
We moved away from CSVs to **Parquet** for performance. The pipeline handles:
* **Ingestion:** Merging raw listing data with 90-day behavioral signals (Views, Clicks, Leads).
* **Normalization:** Cleaning dirty data (e.g., mapping `furnished: "Yes"` $\to$ `1`, `completion: "Ready"` $\to$ `completed`).

### B. The AI Brain (`light_brain.pkl`)
* **Model:** XGBoost Classifier (Optimized for Speed).
* **Objective:** Binary Classification (Target: `lead_submission_count > 0`).
* **Key Features:**
    * `smart_popularity_score` (Weighted interaction metric).
    * `price` (Relative to market).
    * `freshness_score` (Time decay).
    * `trust_score` (Agent quality).

### C. The API (`app_7.py`)
A high-performance **FastAPI** service that loads the Data and Brain into **RAM** for sub-50ms inference.

---

## 3. The "Strict but Smart" Logic

A major challenge in Real Estate search is handling messy data (e.g., missing coordinates) without returning empty results. We implemented a robust fallback chain.

### Layer 1: Hard Filters (The "Strict" Part)
We apply strict boolean masks for critical user requirements. If a user asks for "2 Beds", we never show 1 Bed.
* **Filters:** `category_id` (Buy/Rent), `price_range`, `bedrooms`, `bathrooms`, `furnished_status`.

### Layer 2: Geospatial Intelligence (The "Smart" Part)
Users search by **Keyword** (e.g., "Marina"), but computers need **Coordinates**.
1.  **Centroid Lookup:** The engine checks if "Marina" matches a known location centroid.
2.  **Radius Search:** If matched, it calculates Haversine Distance and fetches listings within **3.5km**.

**The Fallback Safety Net:**
* **Scenario:** A listing is in Marina but has `lat: 0.0, lon: 0.0` (bad data).
* **Fix:** The Radius search fails (0 results). The engine detects this and automatically switches to **Text Match** (`WHERE location_name LIKE '%Marina%'`), ensuring the user still gets results.

### Layer 3: AI Re-Ranking (Learning-to-Rank)
Once Layer 1 & 2 narrow the pool (e.g., from 72,000 $\to$ 200 listings), the AI takes over.
* **Feature Extraction:** The engine builds a feature vector for the 200 candidates.
* **Inference:** The XGBoost model predicts a score ($0.0 \to 1.0$) representing "Lead Probability".

**Scoring Logic:**
$$
\text{Final Score} = (\text{AI Probability} \times 100) - (\text{Distance} \times 0.1) + \text{Luxury Boost}
$$

**Result:** A popular, high-quality listing 1km away ranks higher than a stale listing 0.5km away.

---

## 4. Deployment Status (Phase 1)

| Feature | Status | Notes |
| :--- | :---: | :--- |
| **Basic Search** | Live | Filters by Price, Beds, Area working perfectly. |
| **Geo Search** | Live | Includes Radius Search + "Zero Coordinate" Fallback. |
| **AI Ranking** | Live | "Featured" sort uses XGBoost probability. |
| **Price Period** | Hidden | Logic exists in DB but API filter is disabled for stability. |
| **Performance** | High | In-Memory Parquet loads < 1s, Query < 50ms. |

---

## 5. Phase 2 Roadmap: Moving to "Industry Grade"

While Phase 1 is a robust Search Engine, Phase 2 transforms it into a **Personalization Engine**.

### Objective 1: User-Level Personalization
Currently, every user sees the same "Popular" results.
* **The Plan:** Implement a "Shadow Profile" using Redis.
* **Logic:**
    * If User A clicks 3 Villas $\to$ Boost Villa scores by 20%.
    * If User B filters "Price > 5M" $\to$ Tag as "Luxury User" and deprioritize cheap listings.

### Objective 2: Hierarchical Location Search
Currently, "Dubai" search relies on radius or text match.
* **The Plan:** Ingest `full_location_path` (e.g., Dubai - Marina - Princess Tower).
* **Logic:** Use Path Containment (`path LIKE 'Dubai%'`) to officially capture all child communities without relying on distance.

### Objective 3: Rental Period Logic (`price_type`)
* **The Plan:** Expose the `price_period` filter in the API.
* **Logic:**
    * Allow users to toggle "Daily" vs "Yearly".
    * Prevent 500 AED (Daily) listings from cluttering a "Cheap Yearly Rent" search.


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





