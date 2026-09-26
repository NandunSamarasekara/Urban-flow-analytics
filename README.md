# 🚕 Urban Flow Analytics

**An end-to-end data engineering + BI project analyzing 12 months of NYC-style taxi trip data to surface congestion, transit-equity, and fleet-decarbonization insights.**

![Databricks](https://img.shields.io/badge/Databricks-Free%20Edition-FF3621?logo=databricks&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD8)
![H3](https://img.shields.io/badge/Uber%20H3-Geospatial%20Indexing-000000)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)

Built for **DATATHON 2026** — an industrial-style ELT pipeline (Bronze → Silver → Gold) on Databricks, feeding a 5-page interactive Power BI dashboard connected live via a Databricks SQL Warehouse.

---

## 📊 The Dashboard

![Executive Overview](reports/screenshots/01_executive_overview.png)

Three business questions drive the whole project:

| Area | Key Metrics |
|---|---|
| 🚦 **Congestion & Bottleneck Analysis** | Travel Time Variability Index (TTVI), Congestion Slowdown % |
| 🚉 **First-/Last-Mile Transit Integration** | Transit Hub Connectivity Rate, Avg. Distance to Nearest Hub |
| 🔋 **Fleet Decarbonization & EV Readiness** | Avg. Daily Distance per Vehicle, % Vehicle-Days under EV Range |

See all 5 dashboard pages in [`reports/screenshots/`](reports/screenshots/).

---

## 🏗️ Architecture

```mermaid
flowchart LR
    subgraph Source
        A[12 monthly trip CSVs<br/>~5GB, Google Drive]
    end
    subgraph Databricks["Databricks Lakehouse (Unity Catalog)"]
        B[("🥉 Bronze<br/>raw_files volume")]
        C[("🥈 Silver<br/>trips_clean, trips_enriched")]
        D[("🥇 Gold<br/>5 dims + 3 facts<br/>star schema")]
    end
    subgraph BI["Power BI"]
        E["Live SQL Warehouse connection<br/>(Import mode)"]
        F["5-page dashboard"]
    end
    A -->|gdown, direct-to-volume| B
    B -->|PySpark: cleaning,<br/>H3 hub-matching,<br/>vehicle assignment| C
    C -->|PySpark: business<br/>metric design| D
    D --> E --> F
```

## 📁 Repository Structure

```
urban-flow-analytics/
├── README.md              ← you are here
├── DATASET.md              ← source data, schema, scale, known data-quality caveats
├── WALKTHROUGH.md           ← the build story, chronologically, bugs and all
├── TECHNIQUES.md            ← the data engineering & BI techniques used, explained
├── notebooks/                ← Databricks notebooks (01 → 04, Bronze to Gold)
└── reports/
    └── screenshots/          ← all 5 dashboard pages, exported as PNG
```

## 🧰 Tech Stack

- **Compute:** Databricks (Free Edition), PySpark, Spark SQL
- **Storage:** Delta Lake, Unity Catalog (bronze / silver / gold schemas)
- **Geospatial:** Uber H3 hierarchical geohashing (resolution 8, k-ring search)
- **BI:** Power BI Desktop, live Databricks SQL Warehouse connector, DAX

## 📖 Read More

- **[DATASET.md](DATASET.md)** — where the data came from, what's in it, and what's wrong with it (documented honestly)
- **[WALKTHROUGH.md](WALKTHROUGH.md)** — the full build narrative: decisions, dead ends, and fixes
- **[TECHNIQUES.md](TECHNIQUES.md)** — deep dives into the engineering techniques: medallion architecture, H3 geohashing, dimensional modeling, and the Power BI/DAX patterns used

## 👤 Author

**Nandun Neelaka Samarasekara**
[GitHub](https://github.com/NandunSamarasekara) · [LinkedIn](https://www.linkedin.com/in/nandun-samarasekara-5564162b8/) · [Medium](https://medium.com/@nandunneelaka)

