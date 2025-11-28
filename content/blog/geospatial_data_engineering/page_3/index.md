---
title: "Part 3: The Data Pipeline – Serverless ETL and Analytics with DuckDB & dbt"
date: 2025-11-28
authors:
  - andreshernandez
_build:
  list: never
  render: always
tags:
  - DuckDB
  - dtb
  - Geospatial
---

Welcome to the finale of our series on building a cloud-native geospatial platform. In Part 1, we engineered a robust API with FastAPI and PostGIS. In Part 2, we hardened our infrastructure with Docker, Helm, and CI/CD. We have built a high-performance engine and a state-of-the-art chassis, but an engine is useless without fuel. In the geospatial world, that fuel is data—often terabytes of it.

In this final installment, we are solving the data engineering challenge. We will reject the traditional approach of "downloading the whole world" into our database. Instead, we will build an On-Demand Data Pipeline. We will harness DuckDB to stream building footprints from S3 directly into PostGIS only when requested, and we will deploy dbt (data build tool) as a Kubernetes Job to automate complex spatial transformations. By the end of this post, you won't just have an app; you will have a self-sustaining data ecosystem.

## Serverless Geospatial ETL: Ingesting Open Buildings with DuckDB and FastAPI
In the geospatial world, access to high-quality building footprints is a game-changer. The Open Buildings dataset provides this at a global scale, but managing terabytes of global footprint data in a standard PostGIS database is expensive and inefficient.

In this update to geoproject, we are implementing an "On-Demand" data pipeline. Instead of pre-loading the entire world, we will fetch building footprints dynamically from S3 using DuckDB only when a user requests a specific area. This approach minimizes storage costs while keeping the application responsive.

We chose the [Google-Microsoft-OSM Open Buildings - combined by VIDA](https://source.coop/vida/google-microsoft-osm-open-buildings/README.md) because of his rich dataset, by country aggregation, and Parquet format that makes it ideal for our use case. 

### The Architecture: DuckDB as the Engine
We are introducing a new architectural component: a transient ETL (Extract, Transform, Load) worker powered by DuckDB. DuckDB excels at reading Parquet files directly from S3 and can perform spatial operations on the fly.

The flow is simple:

- Check Local: The API first checks our local PostGIS database for footprints in the requested area.
- Fetch Remote: If data is missing, it triggers DuckDB to read the specific country's Parquet file from the Open Buildings S3 bucket.
- Ingest: DuckDB filters the data spatially and inserts it directly into PostGIS.
- Serve: The API returns the newly ingested data to the user.

#### Step 1: Intelligent Country Lookup
The Open Buildings dataset is partitioned by country. To fetch data efficiently, we first need to know which country the user's Area of Interest (AOI) falls into.

We implemented a utility in [data_pipeline/utils.py](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/data_pipeline/utils.py) using geopandas and country_converter. It performs a spatial join between the user's requested geometry and a low-resolution map of world boundaries to determine the ISO country code.

```python
# data_pipeline/utils.py

def get_country_from_aoi(aoi_wkt: str) -> str:
    """
    Determines the country for an AOI using pure Python (GeoPandas).
    """
    world_gdf = get_world_boundaries()
    
    # ... logic to load AOI ...

    joined = gpd.sjoin(aoi_gdf, world_gdf, how="inner", predicate="intersects")
    if not joined.empty:
        country_code = "".join(joined.WB_A3.values)
        if country_code in COUNTRY_LIST:
            return country_code
    raise ValueError("No country found for AOI.")
```

### Step 2: The DuckDB Pipeline
This is where the magic happens. In [data_pipeline/ingest_building.py](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/data_pipeline/ingest_building.py), we configure DuckDB to connect to S3 and our Postgres database simultaneously. We use the httpfs and postgres extensions to bridge the two worlds.

Notice how we construct a single SQL query that reads from S3 (read_parquet), filters by geometry (ST_Intersects), and inserts into PostGIS (INSERT INTO prod_db...) in one go.

```python
# data_pipeline/ingest_building.py

def query_open_buildings(filter_polygon_wkt: str, confidence: float = 0.75) -> None:
    # ... setup connection ...
    
    # 1. Attach your Production PostGIS DB
    con.sql(f"ATTACH '{pg_conn_string}' AS prod_db (TYPE POSTGRES);")
    
    # 2. Define the S3 URL for the specific country
    s3_url = f"{OPEN_BUILDINGS_BUCKET}/country_iso={country_iso}/{country_iso}.parquet"
    
    # 3. The "ETL" Query: Read S3 -> Filter -> Write to PostGIS
    query = f"""
        INSERT INTO prod_db.public.building_footprints (geom, confidence, area_meters)
        SELECT
            ST_AsWKB(geometry) as geom,
            confidence,
            area_in_meters as area_meters
        FROM read_parquet('{s3_url}')
        {spatial_filter} AND (confidence > {confidence});
    """
    con.sql(query)
```

### Step 3: Service Layer Orchestration
Finally, we wire this logic into our API service in [geoproject/services/building_footprints.py](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/geoproject/services/building_footprints.py). The service acts as a cache controller. It attempts to find data in the local database first. If query.first() is None, it calls the ingestion function we defined above, effectively "hydrating" the cache on demand.

```python
# geoproject/services/building_footprints.py

def get_building_footprints(
    building_footprint_request: BuildingFootprintRequest,
    db: Session = Depends(get_session),
    get_open_buildings: IngestionFunction = Depends(get_open_buildings_dependency),
) -> list[BuildingFootprint]:
    
    # ... setup geometry ...

    # 1. Try to find data locally
    query = db.query(DBBuildingFootprint).filter(
        DBBuildingFootprint.geom.ST_Within(input_geom)
    )
    
    # 2. If missing, trigger ingestion from S3
    if query.first() is None:
        get_open_buildings(geometry.wkt)
        
        # 3. Re-query the database
        query = db.query(DBBuildingFootprint).filter(
            DBBuildingFootprint.geom.ST_Within(input_geom)
        )

    return build_response(query.all())
```

By combining DuckDB's fast columnar processing with PostGIS's robust indexing, we have created a highly efficient hybrid data architecture. Our application stays lightweight, but it has access to a petabyte-scale dataset the moment a user asks for it.

### Step 4. Testing the ingestion pipeline via Postman

We have created a new table in the database that store the footprints that are not already queried before from the VIDA dataset. Check in our alembic models that we updated our database to be able to store the data described before [6205ce21172e_added_building_footprints_table.py](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/geoproject/alembic/versions/6205ce21172e_added_building_footprints_table.py).

Also we added a new route to return the data from the footprints, via extraction or already stored in the `building_footprints` table. And we added in a new route in the [geoproject/api/v1/building_footprints.py](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/geoproject/api/v1/building_footprints.py) files.

```python
@router.post("/", response_model=BuildingFootprints)
def get_all_building_footprints(
    building_footprint_request: BuildingFootprintRequest,
    db: Session = Depends(get_session),
    get_open_buildings: IngestionFunction = Depends(get_open_buildings_dependency),
):
    list_footprints = get_building_footprints(
        building_footprint_request=building_footprint_request,
        get_open_buildings=get_open_buildings,
        db=db,
    )
    return BuildingFootprints(building_footprints=list_footprints)
```

This configuration allow us to test via postman our new extraction layer locally. Via Post request to the `http://127.0.0.1:8000/api/v1/building_footprints` endpoint with a payload 

```json
{
  "geometry": {
        "type": "Polygon",
        "coordinates":[
          [
            [
              14.48201087545425,
              35.89692421566235
            ],
            [
              14.48472507025528,
              35.89692421566235
            ],
            [
              14.48472507025528,
              35.903409043471314
            ],
            [
              14.48201087545425,
              35.903409043471314
            ],
            [
              14.48201087545425,
              35.89692421566235
            ]
          ]
        ]
    }
}
```

we get the footprints from the VIDA dataset that intersect the AOI in the payload. Take into account if the data is not in the local database it will trigger the extraction routine, which could lead to extended times depending on the dataset. I suggest to choose a small country like Malta to be able to get results quickly the first time. However, once the download is done, the data will be on your local PostGIS database and you can retrieve it almost instanly, the next time you request the same area. 

The example results is like the following:

```json
{
    "building_footprints": [
        {
            "location_id": 3,
            "confidence": null,
            "area_meters": 37.4074,
            "geometry": {
                "type": "Polygon",
                "coordinates": [
                    [
                        [
                            14.482133508719631,
                            35.9016509699792
                        ],
                        [
                            14.482035767395956,
                            35.90164850967939
                        ],
                        [
                            14.482037235371141,
                            35.901610330327436
                        ],
                        [
                            14.482134973186817,
                            35.90161279608117
                        ],
                        [
                            14.482133508719631,
                            35.9016509699792
                        ]
                    ]
                ]
            }
        },
        {
            "location_id": 4,
            "confidence": null,
            "area_meters": 279.1295,
            "geometry": {
                "type": "Polygon",
                "coordinates": [
                    [
                        [
                            14.482784014833047,
                            35.90190835361447
                        ],
                        [
                            14.482975479364274,
                            35.90191504004688
                        ],
                        [
                            14.482967745969153,
                            35.90206033728772
                        ],
                        [
                            14.482776293968895,
                            35.90205365130524
                        ],
                        [
                            14.482784014833047,
                            35.90190835361447
                        ]
                    ]
                ]
            }
        }
    ]
}
```

Testing of the new endpoint is also included in the corresponding repo test folder: [tests/api/v1
/test_building_footprints.py
](https://github.com/geotechblogs/geospatial_project/blob/6bf485ef7c0e578a62a5edc03b8878c19f16c2f7/tests/api/v1/test_building_footprints.py)

## Mastering Geospatial Transformations: Integrating dbt with PostGIS

In our previous section, we built a robust ingestion pipeline using DuckDB to load building footprints into PostGIS. However, raw data is rarely ready for immediate analysis or frontend display. It often needs cleaning, enrichment, and standardization.

In this tutorial, we are adding dbt (data build tool) to our stack. dbt allows us to write data transformation logic using simple SQL SELECT statements, while handling the complex orchestration of creating tables and views for us. We will set up a dbt project that takes our raw building footprints and calculates useful spatial metadata—like bounding boxes—automatically.

This tool will be stored in a different repo following good practices, the code for the dbt transformation pipeline is independent from the main app, but they will be deployed together via a kubernetes job, which will connect to the database via kubernetes secret credentials. Github repo for the dbt_geoproject in the following link: [dbt_geoproject](https://github.com/geotechblogs/dbt_geoproject) 

### 1. Setting Up dbt with Poetry
First, we need to add dbt to our python environment. Since we are using Poetry, we define our dependencies in pyproject.toml. We are installing dbt-core along with the dbt-postgres adapter to communicate with our database.

```ini,toml
[project]
name = "dbt-geospatial-project"
version = "0.1.0"
# ...
dependencies = [
    "dbt (>=1.0.0.40.11,<2.0.0.0.0)",
    "dbt-postgres (>=1.9.1,<2.0.0)"
]
```

### 2. Defining Sources and Staging Models
The first step in any dbt project is telling dbt where your data lives. We do this using a sources.yml file. Here, we define our public schema and the building_footprints table we populated in the previous tutorial.

```yaml
# models/sources.yml
version: 2

sources:
  - name: public
    database: postgres
    schema: public
    tables:
      - name: building_footprints
```

Next, we create a Staging Model. This is a 1-to-1 view of our raw source. It serves as a consistent entry point for all downstream transformations, protecting us if the raw table structure changes later.

```yaml
-- models/staging/stg_raw_building_footprints.sql
SELECT
    id,
    geom,
    confidence,
    area_meters
FROM 
    {{ source('public', 'building_footprints') }}
```

### 3. Spatial Enrichment: Calculating Bounding Boxes
Now for the powerful part: combining dbt's workflow with PostGIS functions.

We often need the bounding box (bbox) of a geometry for frontend applications or spatial indexing. Instead of calculating this on the fly during every API request, we can pre-calculate it using an Intermediate Model.

In models/intermediate/int_building_enrichment.sql, we use PostGIS functions like ST_Envelope, ST_XMin, and ST_YMax to extract the coordinates of the bounding box for every building.

```sql
-- models/intermediate/int_building_enrichment.sql
SELECT
    id,
    confidence,
    area_meters,
    geom,
    -- Calculate the bounding box polygon
    ST_Envelope(geom) AS bbox_geom,
    -- Extract specific coordinates for frontend use
    ST_XMin(ST_Envelope(geom)) AS bbox_xmin,
    ST_YMin(ST_Envelope(geom)) AS bbox_ymin,
    ST_XMax(ST_Envelope(geom)) AS bbox_xmax,
    ST_YMax(ST_Envelope(geom)) AS bbox_ymax

FROM 
    {{ ref('stg_raw_building_footprints') }}
WHERE
    geom IS NOT NULL
```

Notice the use of `{{ ref('...') }}`. This tells dbt to depend on the staging model we created in the previous step, establishing a dependency graph.

### 4. Configuring the Project
Finally, we tie it all together in the dbt_project.yml. We configure our project name and tell dbt how to materialize our models.

We set our default materialization to view, but for our intermediate models (where the heavy lifting happens), we set them to materialize as table. This ensures the expensive PostGIS calculations are run once during the build, making subsequent queries lightning fast.

```yaml
# dbt_project.yml
name: 'dbt_geospatial_project'
version: '1.0.0'
profile: 'dbt_geospatial_project'

# ... paths config ...

models:
  dbt_geospatial_project:
    +materialized: view

    intermediate:
      +materialized: table
```
We have successfully integrated dbt into our geospatial stack! We now have a structured way to transform raw data into enriched, analytical-ready tables using standard SQL and PostGIS.

## Automating Geospatial Analytics: Running dbt on Kubernetes with Helm
We have built a powerful ingestion engine using DuckDB and FastAPI, and our PostGIS database is filling up with raw building footprint data. But raw data is rarely ready for analysis. It needs to be cleaned, transformed, and aggregated.

In the data world, dbt (data build tool) is the standard for these transformations. In this update, we are containerizing our dbt project and deploying it to our Kubernetes cluster as a Job. This allows us to run heavy geospatial transformations automatically and reliably in the cloud.

### 1. Dockerizing dbt with Poetry
Running dbt locally is easy (dbt run). Running it inside a Docker container, especially when managing dependencies with Poetry, requires a bit of finesse.

We created a dedicated Dockerfile for our dbt worker. We faced a specific challenge here: making Poetry play nicely with the system's Python environment so that dbt is executable globally without needing to activate a virtual environment manually inside the container.

Here is how we solved it. We disabled virtual environment creation and manually appended the poetry path to PYTHONPATH to ensure Python can find our installed packages:

```Dockerfile
# Dockerfile

# CRITICAL FIX: Ensure poetry installs into the global path (no venv)
ENV POETRY_VIRTUALENVS_CREATE=false
WORKDIR /dbt

# CRITICAL FIX B: Find the Poetry environment's site-packages path 
# and add it to the global PYTHONPATH.
RUN export PYTHONPATH="$(poetry env info --path)/lib/python3.11/site-packages:$PYTHONPATH"

# Dynamically generate profiles.yml from environment variables
RUN echo "config: \n\
dbt_geospatial_project: \n\
  target: dev \n\
  outputs: \n\
    dev: \n\
      type: postgres \n\
      host: \"{{ env_var('DBT_HOST') }}\" \n\
      user: \"{{ env_var('DBT_USER') }}\" \n\
      dbname: \"{{ env_var('DBT_NAME') }}\" \n\
      port: 5432 \n\
      schema: public \n\
      threads: 4" > /dbt/profiles.yml

ENTRYPOINT ["dbt"]
```

We have built a powerful ingestion engine using DuckDB and FastAPI, and our PostGIS database is filling up with raw building footprint data. But raw data is rarely ready for analysis. It needs to be cleaned, transformed, and aggregated.

In the data world, dbt (data build tool) is the standard for these transformations. In this update, we are containerizing our dbt project and deploying it to our Kubernetes cluster as a Job. This allows us to run heavy geospatial transformations automatically and reliably in the cloud.

1. Dockerizing dbt with Poetry
Running dbt locally is easy (dbt run). Running it inside a Docker container, especially when managing dependencies with Poetry, requires a bit of finesse.

We created a dedicated Dockerfile for our dbt worker. We faced a specific challenge here: making Poetry play nicely with the system's Python environment so that dbt is executable globally without needing to activate a virtual environment manually inside the container.


Here is how we solved it. We disabled virtual environment creation and manually appended the poetry path to PYTHONPATH to ensure Python can find our installed packages:

```Dockerfile
# Dockerfile

# CRITICAL FIX: Ensure poetry installs into the global path (no venv)
ENV POETRY_VIRTUALENVS_CREATE=false
WORKDIR /dbt

# CRITICAL FIX B: Find the Poetry environment's site-packages path 
# and add it to the global PYTHONPATH.
RUN export PYTHONPATH="$(poetry env info --path)/lib/python3.11/site-packages:$PYTHONPATH"

# Dynamically generate profiles.yml from environment variables
RUN echo "config: \n\
dbt_geospatial_project: \n\
  target: dev \n\
  outputs: \n\
    dev: \n\
      type: postgres \n\
      host: \"{{ env_var('DBT_HOST') }}\" \n\
      user: \"{{ env_var('DBT_USER') }}\" \n\
      dbname: \"{{ env_var('DBT_NAME') }}\" \n\
      port: 5432 \n\
      schema: public \n\
      threads: 4" > /dbt/profiles.yml

ENTRYPOINT ["dbt"]
```

Why this matters: This setup allows us to inject database credentials (DBT_HOST, DBT_USER) at runtime via Kubernetes secrets, rather than hardcoding them into a file.

### 2. Deploying Transformations as Kubernetes Jobs
Unlike our FastAPI service, which needs to be running 24/7, dbt transformations are batch processes—they start, do the work, and finish. In Kubernetes, the perfect resource for this is a Job.

We created a new Helm chart dbt_geoproject_chart to manage this. The core of this chart is dbt-job.yaml. It spins up a pod, runs the dbt command, and then shuts down (with a TTL to clean up the history).

Notice how we map the Kubernetes Secrets (created by our Postgres chart) directly to the environment variables our Dockerfile expects:

```yaml
# dbt_geoproject_chart/templates/dbt-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "dbt-geospatial-jobs.fullname" . }}-{{ .Values.job.name }}-{{ randAlphaNum 5 | lower }} 
spec:
  template:
    spec:
      containers:
      - name: dbt-runner
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        env:
          - name: DBT_USER
            valueFrom:
              secretKeyRef:
                name: postgres-db-secret
                key: PG_USER
          - name: DBT_HOST
            value: {{ .Values.postgres.host | quote }}
          - name: DBT_NAME
            value: {{ .Values.postgres.dbname | quote }}
        
        # We can override the command in values.yaml, e.g., ["test"] or ["run"]
        args: {{ toYaml .Values.job.args | nindent 8 }}
```

### 3. Configuration Cleanup
Finally, we made a small but important tweak to our dbt source configuration. In models/sources.yml, we removed the hardcoded database: postgres line.

```yaml
# dbt_geospatial_project/models/sources.yml
sources:
  - name: public
    # database: postgres  <-- Removed this
    schema: public
    tables:
      - name: building_footprints
```

By removing this, we allow the profiles.yml (generated in our Dockerfile) to dictate which database name to use. This makes our project portable across different environments (e.g., geoproject_dev vs geoproject_prod) without changing the code.

### 4. Running our dbt pipeline

Once we have all the configuration ready, we need only to run our kubernetes job in our cluster. To do so, we need to again build the Docker image and push it to the docker repository as we did in Part 2. and run the helm install command. 

However, in this case there is an extra step, we rely on kubernetes secrets to set the postgres database credentials, therefore I share the command, or you can also create a secrets.yaml file with the correct configuration.

to get the password assigned in your cluster you can run the following command

```bash
kubectl get secret postgres-db-secret -o yaml
```

The output:
```bash
apiVersion: v1
data:
  PG_DBNAME: <PG_DBNAME in base64 format>
  PG_PASSWORD: <PG_PASSWORD in base64 format>
  PG_USER: <PG_USER in base64 format>
kind: Secret
metadata:
  creationTimestamp: "2025-11-27T18:58:57Z"
  name: postgres-db-secret
  namespace: default
  resourceVersion: "259476"
  uid: f5f85cdf-9c48-454a-b235-4a3697f073bb
type: Opaque
```
Then transfrom the PG_PASSWORD value from base64 to plain text and run the following command to setup your kubernetes secrets and set up you dbt job correctly.

```bash
kubectl create secret generic postgres-db-secret --from-literal=PG_USER=geoproject_user --from-literal=PG_PASSWORD=<YOUR-PASSWORD-PLAIN-TEXT> --from-literal=PG_DBNAME=geoproject_db
```

Then you can run your helm job, by running the helm command
```bash
helm install dbt-jobs-release ./dbt_geoproject_chart
```

The output
```bash
NAME: dbt-jobs-release
LAST DEPLOYED: Fri Nov 28 21:20:58 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

And if you run
```bash
kubectl get jobs
```

you will get something like this: 
```
NAME                                                        STATUS     COMPLETIONS   DURATION   AGE
dbt-jobs-release-dbt-geospatial-jobs-dbt-manual-run-vfxgt   Complete   1/1           10s        12s
```

The final test is to check the database, if you get a new table named `int_building_enrichment` with the data from `building_footprints`, but with the data from the bounding box, you were able to run a transformation job offline, that improve your extracted data with some added values that make it more suitable for end users.

```
| id | confidence | area_meters | geom | bbox_geom | bbox_xmin | bbox_ymin | bbox_xmax | bbox_ymax |
|3306686|0.8229|701.6466|POLYGON ((-73.6916646 3.8261686,...))|POLYGON ((-73.6920691 3.8260269000000005, ...))|-73.6920691|3.8260269000000005|-73.6916646|3.8262899000000004|
```

## Final Thoughts

Over the course of these three posts, we have journeyed from a simple pip install on a local machine to a fully orchestrated cloud platform. We have replaced fragile scripts with FastAPI services, swapped manual deployments for Helm charts, and evolved from ad-hoc SQL queries to version-controlled dbt models. You have successfully built geoproject—a system that can ingest global data on-demand, self-heal in Kubernetes, and verify its own logic through CI/CD.

You are no longer just writing geospatial code; you are architecting geospatial systems.

However, a software platform is never truly "finished." The foundation you have laid here is ready for the next layer of complexity. You might explore adding a frontend visualization with Deck.gl, integrating raster data processing with Cloud Optimized GeoTIFFs (COGs), or securing your endpoints with OAuth2. The map is open, the infrastructure is solid, and the tools are in your hands. Now, go build.

### References

- Want to see the previous Blog [Part 2: The Infrastructure – Dockerizing GDAL and Deploying with Helm]({{< ref "/blog/geospatial_data_engineering/page_2" >}})
- or go to the main blog: [Building a Cloud-Native Geospatial Platform: From Localhost to Kubernetes]({{< ref "/blog/geospatial_data_engineering/main_page" >}})


All code and documented commits for the application:
- [Geotechblogs geoproject official repo](https://github.com/geotechblogs/geospatial_project)
- [Geotechblogs dtb_configuration official repo](https://github.com/geotechblogs/dbt_geoproject)
