# Project Checklist

This is a overview/brain-dump of the tasks that I have thought of that I will most likely need to do. I've probably missed some components, but I'll get to those when I actually get around to implementing each part. High-level, it is EDA -> data engineering pipeline -> modeling -> MLops -> serving.

## 1: Preliminary EDA

- get acquainted with the different APIs and data sources
    - [x] NASA FIRMS API ->`prelim_eda/eda_firms_wfigs_current.ipynb`
    - [x] WFIGS perimeters -> `prelim_eda/eda_firms_wfigs_current.ipynb`
    - [x] ICS-209-PLUS -> `prelim_eda/eda_historical_ics209.ipynb`
    - [x] bulk archive download for VIIRS SNPP and NOAA-20 (2018-2023 CA) -> `prelim_eda/eda_bulk_download.ipynb`
    - [x] look at different weather APIs to figure out which to use -> `prelim_eda/eda_weather_api_comparison.ipynb`
    - [] look at different terrain APIs/data sources -> TODO

---

## 2: Data Engineering Pipeline

Plan as of now is to do some form of EDA for each stage in the data engineering pipeline to figure out how it should work, then extract this logic into clean modules. Result of this should be defined input schema and output schema for each stage.

Not every stage necessarily needs to write to the DB. Stages that are expensive or natural restart points (raw ingest, clustering, labeling, feature matrix) probably should, but lighter transforms in between might be fine as in-memory or intermediate parquet. Will figure this out when actually implementing each stage.

- [ ] Stage 0: Raw Data
    - input:
        - VIIRS SNPP and NOAA-20 bulk archive CSVs
        - FIRMS NRT API (live feed for serving pipeline)
        - WFIGS perimeters
        - ICS-209-PLUS
        - terrain data (TBD)
    - output:
        - per-satellite parquet files -> loaded into DB
        - WFIGS perimeters -> loaded into DB
        - ICS-209-PLUS incident records -> loaded into DB
        - terrain data (TBD) -> loaded into DB
- [ ] Stage 1: Preprocessing
    - input:
        - per-satellite parquets from Stage 0
    - output:
        - single merged, cleaned hotspot table -> loaded into DB
    - some items to consider:
        - deduplication
        - column name normalization
        - filtering
- [ ] Stage 2: Spatial Partitioning
    - input:
        - clean hotspot table from Stage 1
    - output:
        - hotspot table with tile_id assigned to each row -> loaded into DB
    - some items to consider:
        - tile size and overlap width
        - fires that straddle tile boundaries
- [ ] Stage 3: ST-DBSCAN Clustering
    - input:
        - per-tile hotspot subsets from Stage 2
    - output:
        - hotspots with cluster_id assigned (-1 = noise) -> loaded into DB
    - some items to consider:
        - parameter selection (spatial radius, time window, min samples)
        - boundary reconciliation across adjacent tiles
- [ ] Stage 4: Cluster Aggregation
    - input:
        - clustered hotspots from Stage 3
    - output:
        - daily cluster snapshot table (one row per cluster per day) -> loaded into DB
    - some items to consider:
        - detection gaps, satellite not passing != fire went out
- [ ] Stage 4b: Weather Enrichment
    - input:
        - daily cluster snapshots with centroid geometry
    - output:
        - weather fields joined to each cluster-day -> loaded into DB
    - some items to consider:
        - which fields are worth fetching (wind, temp, humidity, precip?)
- [ ] Stage 5: Feature Engineering
    - input:
        - daily cluster snapshots + weather data from Stage 4/4b
    - output:
        - feature vector per cluster at observation cutoff -> loaded into DB
    - some items to consider:
        - no future data bleeding into features (leakage)
        - how to handle young clusters with little history
- [ ] Stage 5b: CV Feature Extraction (parallel branch, most likely not doing in first version of project)
    - input:
        - cluster bounding boxes + raw VIIRS thermal imagery
    - output:
        - CV-derived features per cluster -> loaded into DB
- [ ] Stage 6: Labeling
    - input:
        - cluster features + ICS-209-PLUS
    - output:
        - labeled clusters (positive / unlabeled / negative) -> loaded into DB
    - some items to consider:
        - matching strategy, cluster centroid to nearest ICS-209 origin
        - three-way split to avoid forcing ambiguous clusters into negative class
        - timezone differences between data sources
- [ ] Stage 7: Feature Matrix Assembly
    - input:
        - labeled clusters from Stage 6
    - output:
        - X_train, y_train, X_test -> loaded into DB
    - some items to consider:
        - time-based train/test split (no random splits)
        - what to do with unlabeled rows

---

## Pipeline Module Implementation

- Take the decisions and logic found during the EDA of each pipeline stage and create clean python modules. Goal is to have an end-to-end pipeline that takes the raw data and outputs model ready features.

---

## Risk Model

- Bulk of the modeling work to generate the risk scores. The data being worked with will be highly imbalanced so need to consider that. Start with XGBoost and/or Random Forests. Should optimize against false positives.

---

## MLOps

- Much of the infrastrucutre work for the MLops and integrating MLflow.
- experiment tracking and model registry with MLflow
- model promotion through CI/CD: staging pipeline trains and evals a candidate, production pipeline runs on held-out test data, best model gets promoted
- monitoring loop: retroactively label predicted clusters once ground truth arrives, track realized precision, alert if it drops
- retraining triggers: scheduled cadence, performance-based, and manual

---

## API Serving

- create the endpoints for predicitions, something along the lines of `GET /fires`, `GET /fires/{id}`, `GET /fires/{region}`, etc..
- scheduler to poll FIRMS NRT data on ~3h cadence and run the pipeline on incoming data
- containerize with Docker, figure out deployment (Kubernetes or something simpler for v1)
- Redis in front of Postgres for caching, data only changes every ~3h so repeated reads should be cheap

---

## CV Pipeline (optional)

- get raw VIIRS thermal imagery (GEE for prototyping, LAADS DAAC for more control)
- train a fire pixel classifier on top of the raw imagery, bootstrapped from FIRMS detections as labels
- wire up Stage 5b from pipeline and feed CV features back into the modeling
- render false-color composites per cluster for the API response
