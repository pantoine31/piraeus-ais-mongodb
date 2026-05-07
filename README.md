# piraeus-ais-mongodb

Modeling and analysis of **enriched ship trajectories** from the Piraeus AIS dataset (December 2018) in **MongoDB**, with spatio-temporal & 2dsphere indexes, an ETL pipeline for spatio-temporal weather enrichment from NOAA, and aggregation queries.

Semester project for the course *Data Management for Relational and Non-relational Databases*.
MSc Program **Information Systems & Services — Specialization: Big Data and Analytics**, Department of Digital Systems, **University of Piraeus** (2025–2026).
Course instructor: **C. Doulkeridis**.

**Authors:** Antonios Papakonstantinou, Dimitrios Kostiris.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Installation & Setup](#-installation--setup)
- [Data Pipeline (ETL)](#-data-pipeline-etl)
- [Database Schema (MongoDB)](#-database-schema-mongodb)
- [Indexes](#-indexes)
- [Queries](#-queries)
- [Backup / Restore](#-backup--restore)
- [Deliverables](#-deliverables)

---

## 🎯 Overview

AIS (Automatic Identification System) data is large-scale spatio-temporal data. The goal of this project is to move beyond the simple storage of individual **AIS records** towards modeling full **trajectories** as unified document objects in MongoDB, so that complex queries combining spatial, temporal, weather and annotation filters can be served **efficiently** and **scalably**.

The dataset originates from: **Tritsarolis, Kontoulis & Theodoridis** — *“The Piraeus AIS dataset for large-scale maritime data analytics”* ([Zenodo: 6323416](https://zenodo.org/records/6323416)).

---

## 📁 Repository Structure

```
piraeus-ais-mongodb/
├── README.md                      ← this file
├── data/
│   └── ais_dec2018_enriched.csv   ← final enriched dataset (produced by the ETL)
├── notebooks/
│   └── etl_pipeline.ipynb         ← ETL: Static+Dynamic AIS → join with NOAA → enrichment
├── scripts/
│   ├── build_trajectories.js      ← aggregation that builds trajectories from ais_points
│   ├── create_indexes.js          ← creation of all indexes
│   └── mongoimport.sh             ← imports the CSV into the ais_points collection
├── queries/
│   ├── q1_complex_spatio_weather.js   ← complex query (A2.1)
│   └── q2_count_in_box.js             ← aggregation count over a spatio-temporal box (A2.2)
├── experiments/
│   ├── benchmark.py               ← timing measurements with/without indexes, varying sizes
│   └── plots/                     ← scalability plots
├── report/
│   └── technical_report.pdf       ← technical report (ACM 2-column template)
├── slides/
│   └── presentation.pptx          ← 10-slide presentation
└── docker-compose.yml             ← MongoDB 7 container (mongo-bda)
```

---

## 🛠 Prerequisites

- **Docker** ≥ 24 and **Docker Compose**
- **Python** ≥ 3.10 (for the ETL)
- **MongoDB Database Tools** (`mongoimport`, `mongodump`, `mongorestore`) — installed locally or accessed through the container
- (optional) **MongoDB Compass** for visual inspection

Python dependencies:

```bash
pip install pandas numpy pymongo scipy tqdm pyarrow
```

---

## 🚀 Installation & Setup

### 1. Clone the repository

```bash
git clone https://github.com/<your-user>/piraeus-ais-mongodb.git
cd piraeus-ais-mongodb
```

### 2. Start the MongoDB container

The `docker-compose.yml` defines a container named **`mongo-bda`** (MongoDB 7) with a persistent volume on port 27017:

```bash
docker compose up -d
docker ps   # verify that mongo-bda is running
```

Connect from the host:

```bash
mongosh "mongodb://localhost:27017"
use piraeus_ais
```

### 3. Download the raw dataset

From Zenodo: <https://zenodo.org/records/6323416>
Place the files (dynamic AIS, static AIS, NOAA weather) under `data/raw/`.

---

## ⚙️ Data Pipeline (ETL)

All steps are implemented in `notebooks/etl_pipeline.ipynb` (developed on Google Colab).

1. **Time-window selection:** December 2018 — chosen as a balance between data volume and processing time (the full dataset is over 244M records).
2. **Dynamic + Static AIS unification:** join via `vessel_id` to attach `country` and `shiptype` to each position record.
3. **Spatio-temporal enrichment with NOAA:** for each AIS record, **nearest-neighbor** matching is applied at the corresponding hour — the spatially closest weather measurement point is selected and its `TMP`, `PRMSL`, `RH`, `WSPD`, `APCP` variables are copied.
4. **Quality control:** the distance `weather_dist_m` to the nearest weather station is also stored, so that the reliability of the matching can be filtered downstream.
5. **Final CSV generation:** `data/ais_dec2018_enriched.csv` — this is the file imported into MongoDB.

Final CSV header:

```
timestamp, vessel_id, lon, lat, speed, course, heading,
country, shiptype, TMP, PRMSL, RH, WSPD, APCP, weather_dist_m
```

### Importing into MongoDB

```bash
bash scripts/mongoimport.sh
# or equivalently:
mongoimport --uri "mongodb://localhost:27017/piraeus_ais" \
            --collection ais_points \
            --type csv --headerline \
            --file data/ais_dec2018_enriched.csv
```

### Building the trajectories

```bash
mongosh "mongodb://localhost:27017/piraeus_ais" scripts/build_trajectories.js
```

The script groups records by `(vessel_id, day)` and produces the `trajectories` collection with the structure described below.

---

## 🗂 Database Schema (MongoDB)

Two layers of information per trajectory:

**Document-level** — stable metadata (`vessel_id`, `country`, `shiptype`, `start_time`, `end_time`).
**Point-level** — a `points[]` array, where each point holds a timestamp, a GeoJSON `Point`, dynamic kinematic attributes, and an embedded `weather` object.

```json
{
  "_id": ObjectId("..."),
  "trajectory_id": "8f525a1a...__2018-12-01",
  "vessel_id":     "8f525a1a78db6a0cf3411f9e88cc56a7d74b5525...",
  "country":       "Greece",
  "shiptype":      60,
  "start_time":    ISODate("2018-12-01T00:00:33Z"),
  "end_time":      ISODate("2018-12-01T23:58:12Z"),
  "points": [
    {
      "t":       ISODate("2018-12-01T00:00:33Z"),
      "loc":     { "type": "Point", "coordinates": [23.55916, 37.96016] },
      "speed":   0.1,
      "course":  198.0,
      "heading": 198.0,
      "weather": {
        "TMP":   279.92,
        "PRMSL": 102082.05,
        "RH":    80.81,
        "WSPD":  1.48,
        "APCP":  0.0
      },
      "weather_dist_m": 6406.18
    }
  ]
}
```

Collections that are created:
- `ais_points` — flat staging collection used during raw import.
- `trajectories` — the main collection holding the enriched trajectory documents.
- `trajectory_annotations_clean` — annotations from the original dataset (cleaned).

---

## 🔎 Indexes

Created via `mongosh < scripts/create_indexes.js`:

| Index | Fields | Purpose |
|---|---|---|
| `_id_` *(default)* | `_id` | Document uniqueness |
| `trajectory_id_1` *(unique)* | `trajectory_id` | Prevents duplicate trajectories (vessel × day) |
| `vessel_id_1_start_time_1` | `{vessel_id: 1, start_time: 1}` | Filtering by vessel and time interval |
| `idx_time_range` | `{start_time: 1, end_time: 1}` | Time-range queries at trajectory level |
| `idx_points_loc` | `points.loc` *(2dsphere)* | Spatial queries (`$geoWithin`, `$near`) over individual points |

The **`2dsphere`** index on `points.loc` is the most critical one for the spatio-temporal queries of A2.

---

## 🔬 Queries

### Query 1 (A2.1) — Most complex

`queries/q1_complex_spatio_weather.js`

**Scenario:** For December 2018, find the **Top-5 trajectories with the most “windy-in-box” points**, where a point qualifies if it simultaneously:
- lies inside the bounding box `lon ∈ [23.4, 23.9]` and `lat ∈ [37.7, 38.1]`,
- has `weather.WSPD ≥ 8`.

Also `$lookup` against `trajectory_annotations_clean` (via `trajectory_id`), return counts and 2 sample points per trajectory. Print an informative message if nothing is found.

**Pipeline (summary):**

```js
db.trajectories.aggregate([
  // 1. Fast pre-filter on trajectories that intersect Dec. 2018
  { $match: {
      start_time: { $lt: ISODate("2019-01-01") },
      end_time:   { $gte: ISODate("2018-12-01") }
  }},
  // 2. Unwind the points array
  { $unwind: "$points" },
  // 3. Per-point filter: bbox + WSPD ≥ 8
  { $match: {
      "points.loc": { $geoWithin: { $box: [[23.4, 37.7], [23.9, 38.1]] }},
      "points.weather.WSPD": { $gte: 8 }
  }},
  // 4. Group per trajectory: count points + keep 2 samples
  { $group: {
      _id: "$trajectory_id",
      vessel_id: { $first: "$vessel_id" },
      windy_in_box_count: { $sum: 1 },
      sample_points: { $push: "$points" }
  }},
  { $addFields: { sample_points: { $slice: ["$sample_points", 2] }}},
  // 5. Enrichment with annotations
  { $lookup: {
      from: "trajectory_annotations_clean",
      localField: "_id",
      foreignField: "trajectory_id",
      as: "annotations"
  }},
  // 6. Sort and Top-5
  { $sort:  { windy_in_box_count: -1 }},
  { $limit: 5 }
]);
```

**Why it is efficient:**
- The `$match` before `$unwind` eliminates trajectories outside December using `idx_time_range`, so we never expand their points arrays.
- `$geoWithin` over `points.loc` is served by the **`2dsphere`** index (`idx_points_loc`).
- `$sort + $limit` operates on at most a few hundred documents (after grouping), so its cost is negligible.
- Using `$slice` for the sample points keeps the final payload small.

### Query 2 (A2.2) — Aggregation count over a spatio-temporal box

`queries/q2_count_in_box.js`

**Scenario:** How many **distinct trajectories** have at least one point inside the spatio-temporal box:
- bbox `[23.5, 37.85] × [23.8, 38.0]`
- time interval `[2018-12-10, 2018-12-20]`.

```js
db.trajectories.aggregate([
  { $match: {
      start_time: { $lt: ISODate("2018-12-20") },
      end_time:   { $gte: ISODate("2018-12-10") }
  }},
  { $unwind: "$points" },
  { $match: {
      "points.t": {
        $gte: ISODate("2018-12-10"),
        $lt:  ISODate("2018-12-20")
      },
      "points.loc": {
        $geoWithin: { $box: [[23.5, 37.85], [23.8, 38.0]] }
      }
  }},
  { $group: { _id: "$trajectory_id" }},
  { $count: "trajectories_in_box" }
]);
```

It leverages both `idx_time_range` and `idx_points_loc`. The first `$match` is important because it discards upfront any trajectory whose temporal extent does not overlap the interval of interest.

---

## 💾 Backup / Restore

**Create a backup:**

```bash
docker exec mongo-bda mongodump --db piraeus_ais --out /data/backup
docker cp mongo-bda:/data/backup ./mongo-backup
```

**Restore:**

```bash
docker cp ./mongo-backup mongo-bda:/data/backup
docker exec mongo-bda mongorestore --db piraeus_ais /data/backup/piraeus_ais
```

---

## 📦 Deliverables

According to the project specification, the compressed `.zip` (<10 MB) submitted on *Lefkippos* contains:

- **(A)** Source code (`notebooks/`, `scripts/`, `queries/`, `experiments/`)
- **(B)** Installation instructions (this README)
- **(C)** Technical report in PDF (`report/technical_report.pdf`) — **ACM 2-column** template
- **(D)** 10-slide presentation (`slides/presentation.pptx`)

A full MongoDB backup is available on request during the project evaluation/presentation.

---

## 👥 Authors

| Name | Email |
|---|---|
| Antonios Papakonstantinou | antonhspap@icloud.com |
| Dimitrios Kostiris | dimkostir@gmail.com |

University of Piraeus — Department of Digital Systems, 2026.
