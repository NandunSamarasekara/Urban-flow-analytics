# Notebooks

Place the four pipeline notebooks here, following the existing naming convention:

- `01_bronze_ingestion.ipynb` — pulling the 12 monthly CSVs from Google Drive into the `raw_files` volume
- `02_silver_layer.ipynb` — `trips_clean` and the `is_valid_trip` validity gate
- `03_reference_data.ipynb` — Part A (H3 hub matching) and Part B (synthetic vehicle assignment) → `trips_enriched`
- `04_gold_layer.ipynb` — the star schema build: 5 dimensions, 3 facts

Export each from Databricks as `.ipynb` (File → Export → IPYNB) or `.html`/`.dbc` if you'd rather include a rendered version.
