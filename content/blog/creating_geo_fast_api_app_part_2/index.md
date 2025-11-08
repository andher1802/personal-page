---
title: 📈 Building a Self-Served Geospatial App Part 2 (FastAPI, Pytest & CI/CD)
summary: .
date: 2025-11-08
authors:
  - andreshernandez
tags:
  - FastAPI
  - CircleCI
  - Docker-compose
  - Geospatial
image:
  caption: 'Geospatial application part 2'
---

In [Part 1](https://andher1802.github.io/personal-page/blog/creating_geo_fast_api_app_part_1/), we built a solid foundation: a robust, containerized ETL pipeline using Docker, PostGIS, and Alembic. We successfully transformed spatial data from a CSV file and loaded it into a database, all managed with a single docker-compose command.

But data isn't useful until you can access it.

In this second and final part, we'll build the rest of the application. We will:

1. Build a high-performance API with FastAPI to serve our geospatial data to the world.
2. Write automated tests with Pytest to ensure our API is reliable and correct.
3. Create a professional CI/CD pipeline with CircleCI to automatically build, test, and validate our entire application on every code change.

``` mermaid
graph TD
    subgraph Data Ingestion
        A[Raw Data Files] --> B(ETL Service)
    end

    subgraph Database
        B --> C[PostGIS Database]
    end

    subgraph API Server
        C --> D(FastAPI Server)
    end

    subgraph Clients
        D --> E[Web Application]
        D --> F[Other Applications/Consumers]
    end

    subgraph CI/CD
        G[Git Repository] --> H(CircleCI)
        H --> I{Tests & Build}
        I --> J[Docker Images]
        J --> K(Deployment)
    end

    subgraph Orchestration
        K --> L(Docker Compose)
        L --> B
        L --> C
        L --> D
    end

    B -- "Reads/Transforms" --> C
    C -- "Queries" --> D
    D -- "Serves Data" --> E
    D -- "Serves Data" --> F

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style C fill:#ccf,stroke:#333,stroke-width:2px
    style D fill:#fcf,stroke:#333,stroke-width:2px
    style E fill:#bfb,stroke:#333,stroke-width:2px
    style F fill:#bfb,stroke:#333,stroke-width:2px
    style G fill:#f9f,stroke:#333,stroke-width:2px
    style H fill:#bbf,stroke:#333,stroke-width:2px
    style I fill:#ccf,stroke:#333,stroke-width:2px
    style J fill:#fcf,stroke:#333,stroke-width:2px
    style K fill:#bfb,stroke:#333,stroke-width:2px
    style L fill:#f9f,stroke:#333,stroke-width:2px
```

This diagram illustrates the complete architecture we're building.

- **Data Flow**: It starts with Raw Data Files being processed by our ETL Service. This service transforms the data and loads it into the PostGIS Database. The FastAPI Server then queries this database, making the data available to end-users, such as a Web Application.

- **Infrastructure & CI/CD**: The entire system is managed by Docker Compose, which orchestrates all the services (ETL, Database, and API). Our development pipeline, powered by CircleCI, automatically pulls code from the Git Repository to run tests and build the necessary Docker Images for deployment.

This setup ensures a clean separation of concerns, from data ingestion to final delivery, all wrapped in a robust, automated environment.


When we're done, we will have a complete, production-ready, and fully-automated system. Let's get started.

### The FastAPI REST API
Building a robust API server with FastAPI could be its own series. However, our goal here is broader: we want to explore testing, CI practices, and multi-container deployment. Therefore, we'll focus on the most relevant components for this application, leaving a deep dive into FastAPI for another time.

For simplicity, our API server will only feature two endpoints for reading data. We'll skip write, update, and delete operations for now.

#### Configuring the Application and Initial Routes
First, let's create a dedicated folder for our server at the root of our project and initialize a new FastAPI application.
``` bash
mkdir server_app # From the root folder of the project
cd server_app
poetry add fastapi uvicorn
touch main.py
```

Now, let's add the basic setup for our FastAPI app in the new main.py file.
```python
from fastapi import FastAPI

app = FastAPI()
app.title = "Geospatial Test Application"
app.version = "0.0.1"


@app.get("/")
async def root():
    return {"message": "Geospatial Test Application running!"}
```

You can test the app by running it with Uvicorn from your terminal:
```bash
uvicorn main:app --reload
```

With the main app running, let's define the routes for our endpoints. We'll create a new routes folder and a file to manage our spatial endpoints.
```bash
mkdir routes
cd routes
touch spatial_endpoints.py
```

Inside `spatial_endpoints.py`, we'll define our two endpoints. For now, they will return a hard-coded dummy response.

```python
from datetime import datetime

from fastapi import APIRouter, Query, status
from fastapi.responses import JSONResponse

locations_router = APIRouter()

now = datetime.now().strftime("%m/%d/%Y, %H:%M:%S")

DUMMY_RESPONSE = {
  "content": [
    {
      "org_id": 1,
      "timestamp": now,
      "geometry": "any geometry",
    },
    {
      "org_id": 2,
      "timestamp": now,
      "geometry": "any geometry",
    }
  ],
  "status_code": status.HTTP_200_OK,
}

@locations_router.get(
    "/locations",
    tags=["locations"],
    response_model=list[dict],
    status_code=status.HTTP_200_OK
)
def get_locations() -> dict:
    response = DUMMY_RESPONSE
    return JSONResponse(
        content=response["content"], status_code=response["status_code"]
    )

@locations_router.get(
    "/locations/",
    tags=["locations"],
    response_model=list[dict],
    status_code=status.HTTP_200_OK,
)
def get_locations_by_orgid(
    orgid: int = Query(ge=1, description="orgid for locations to search.")
) -> dict:
    response = DUMMY_RESPONSE
    return JSONResponse(
        content=response["content"], status_code=response["status_code"]
    )
```

Finally, we need to include this new router in our main `main.py` application file.
```python
from routes.spatial_endpoints import locations_router
...
app.include_router(locations_router)
```

After these changes, your app should reload. You can now access the `/locations` endpoint in your browser and see the dummy JSON response.

### Configuring the Services for the API Server
Our app is running, but the routes are just returning static data. Let's create a "service" layer to handle the logic of connecting and extracting data from the database. This involves three steps:

1. Adding services to the routes.
2. Configuring the database connection.
3. Defining the data models.

#### Adding Services to Routes
We'll create a new `services` module to separate our business logic from the routing.
```bash
mkdir services # From the server_app folder
touch __init__.py
touch spatial_locations.py
```

The new `services/spatial_locations.py` file will initially just move the DUMMY_RESPONSE logic from the routes file.
```python
from datetime import datetime

from fastapi import status

now = datetime.now().strftime("%m/%d/%Y, %H:%M:%S")

DUMMY_RESPONSE = [
    {
      "org_id": 1,
      "timestamp": now,
      "geometry": "any geometry",
    },
    {
      "org_id": 2,
      "timestamp": now,
      "geometry": "any geometry",
    }
  ]

def read_locations() -> dict:
    return {
        "content": DUMMY_RESPONSE,
        "status_code": status.HTTP_200_OK,
    }

def read_locations_by_orgid(orgid: int) -> dict:
    return {
        "content": DUMMY_RESPONSE,
        "status_code": status.HTTP_200_OK,
    }
```

Now, we update our routes in `routes/spatial_endpoints.py` to call these new service functions instead of containing the logic themselves.

```python
...
from services.spatial_locations import read_locations, read_locations_by_orgid
...
def get_locations() -> dict:
    response = read_locations() # Replace the response with the one read from the services.spatial_locations
    return JSONResponse(
        content=response["content"], status_code=response["status_code"]
    )

...
def get_locations_by_orgid(
    orgid: int = Query(ge=1, description="orgid for locations to search.")
) -> dict:
    response = read_locations_by_orgid(orgid=1) # Replace the response with the one read from the services.spatial_locations
    return JSONResponse(
        content=response["content"], status_code=response["status_code"]
    )
```


The browser response should be identical, but our code is now better structured for the next step.

### Configuring the Database Connection
It's time to connect our services to the database. We'll create a config module to handle the database connection settings.

```bash
mkdir config # From the server_app folder
touch __init__.py
touch database.py
```

The `database.py` file will use SQLAlchemy to create a database engine and session.

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm.session import sessionmaker

DATABASE_URL = "postgresql+psycopg2://username:password@localhost:5432/spatial_data_db" # We will change this later after configuring the docker compose environment
engine = create_engine(DATABASE_URL)
session = sessionmaker(bind=engine)
Base = declarative_base()
```

### Defining Models
To read from our database with SQLAlchemy, we need to define a model that maps to our spatial_data table. We'll create a models folder for this.

```bash
mkdir models # From the server_app folder
touch __init__.py
touch spatial_locations.py
```

The `models/spatial_locations.py` file will define the table structure.

```python
import sqlalchemy as sa
from config.database import Base

class SpatialLocationsModel(Base):
    __tablename__ = "spatial_data"
    org_id = sa.Column(sa.Integer, primary_key=True)
    timestamp = sa.Column(sa.DateTime(timezone=True), server_default=func.now())
    geometry = sa.Column(
        Geometry("GEOMETRY", srid=4326, spatial_index=True), nullable=False
    )
```


With the model, config, and services in place, we can now update `services/spatial_locations.py` to query the database. This is a significant update:

```python
from datetime import datetime

from config.database import session
from fastapi import status
from fastapi.encoders import jsonable_encoder
from models.spatial_locations import SpatialLocationsModel
from shapely import wkb


def format_results(query_results: any) -> dict:
    """
    Format the result of a postgis query to the location table
    and returns as a GeoJSON for geometries.
    """
    result_list = []
    for result in query_results:
        geom = wkb.loads(bytes(result.geometry.data))
        dict_result = {
            "org_id": result.org_id,
            "timestamp": result.timestamp,
            "geometry": geom.__geo_interface__,
        }
        result_list.append(dict_result)
    return result_list

class SpatialLocationService:
    def __init__(self, db) -> None:
        self.db = db

    def get_locations(self) -> list[dict]:
        result_query = self.db.query(SpatialLocationsModel).all()
        return result_query

    def get_locations_by_orgid(self, org_id: int) -> list[dict]:
        result_query = (
            self.db.query(SpatialLocationsModel)
            .filter(SpatialLocationsModel.org_id == org_id)
            .all()
        )
        return result_query

def read_locations() -> dict:
    try:
        result_query = SpatialLocationService(session()).get_locations()
        result_list = format_results(result_query)
        return {
                "content": jsonable_encoder(result_list),
                "status_code": status.HTTP_200_OK,
            }
    except Exception as error:
        return {
            "content": f"Something went wrong with error {error}",
            "status_code": status.HTTP_400_BAD_REQUEST,
        }


def read_locations_by_orgid(orgid: int) -> dict:
    try:
        result_query = SpatialLocationService(session()).get_locations_by_orgid(orgid)
        result_list = format_results(result_query)
        return {
                "content": jsonable_encoder(result_list),
                "status_code": status.HTTP_200_OK,
        }
    except Exception as error:
        return {
            "content": f"Something went wrong with error {error}",
            "status_code": status.HTTP_400_BAD_REQUEST,
        }
```

A few key points about this update:
- The `SpatialLocationService` class now handles the database session and queries.
- The `read_locations` functions manage the full `request/response` cycle, including error handling.
- The format_results function is crucial. It reads the database query results and correctly formats the PostGIS geometry field into a GeoJSON structure using shapely.

After this, your endpoints should return live data from the database.

### Adding Testing to the API Server
Software testing is a critical field. While we won't cover every testing technique, we'll set up basic unit tests and integrate them into a CI environment.

Let's create a tests folder in our `server_app` directory.

```bash
mkdir tests # From the server_app folder
touch __init__.py
touch test_services.py
```

Inside `test_services.py`, we'll add a few tests for our service layer.
```bash
import pytest
from config.database import DATABASE_URL
from fastapi import status
from services.spatial_locations import (SpatialLocationService, read_locations,
                                        read_locations_by_orgid)
from sqlalchemy import create_engine
from sqlalchemy.orm.session import sessionmaker


@pytest.fixture
def database_session():
    engine = create_engine(DATABASE_URL)
    return sessionmaker(bind=engine)


def test_read_database(database_session):
    result_query = SpatialLocationService(database_session()).get_locations()
    assert len(result_query) > 0


def test_json_serializable_get_all():
    result_query = read_locations()
    assert isinstance(result_query["content"], list)
    assert len(result_query["content"]) == 19 # hard coded value that will change with our real dataset. Not a very good testing practice.


def test_json_serializable_get_byorgid():
    result_query = read_locations_by_orgid(orgid=1)
    assert isinstance(result_query["content"], list)
    assert len(result_query["content"]) == 1


def test_json_serializable_get_byorgid_bad_request() -> None:
    result_query = read_locations_by_orgid(orgid="ERROR")
    assert result_query["status_code"] == status.HTTP_400_BAD_REQUEST
```

With these tests in place, you can run them locally to validate your service logic.

### Dockerizing the API
Our application runs locally, but our goal is a multi-container setup. We need to create a Dockerfile for our server_app and update our docker-compose.yml.

```Dockerfile
FROM python:3.10

WORKDIR /usr/src/app/geo_app

RUN apt-get update && apt-get install -y libpq-dev && \
    apt-get install -y --no-install-recommends gcc python3-dev

RUN apt-get update && apt-get install -y \
    gdal-bin \
    libgdal-dev \
    build-essential \
    && apt-get clean

# Set environment variables for GDAL
ENV CPLUS_INCLUDE_PATH=/usr/include/gdal
ENV C_INCLUDE_PATH=/usr/include/gdal

RUN pip install fiona

COPY poetry.lock ./
COPY pyproject.toml ./

RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --no-ansi

COPY ./server_app/. ./server_app

WORKDIR /usr/src/app/geo_app/server_app

CMD ["poetry", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

Next, add the new fastapi service to the `docker-compose.yml` file at the root of your project.

```yml
...
  fastapi:
      build:
        context: ./
        dockerfile: ./server_app/Dockerfile
      environment:
        DATABASE_URL: postgresql://username:password@database:5432/spatial_data_db
      depends_on:
        database:
          condition: service_healthy
      ports:
        - 3200:8000
      networks:
        - mynetwork
```

Remember to update `config/database.py` to read the `DATABASE_URL` from the environment variable. With these changes, you can build and run your entire stack:

```bash
docker-compose build
docker-compose up -d
```

Your API should now be running from within its container and accessible on localhost:3200.

### CI through GIT and CircleCI
The final step is setting up a Continuous Integration (CI) pipeline. We'll use CircleCI, configured via a `.circleci/config.yml` file.

```yml
version: 2.1

workspace: &workspace-dir /tmp/workspace


jobs:
  test:
    working_directory: *workspace-dir
    docker:
      - image: cimg/python:3.10
        user: root
        environment:
          DATABASE_URL: postgresql://username:password@localhost:5432/spatial_data_db
      - image: postgis/postgis:13-3.1
        environment:
          POSTGRES_USER: username
          POSTGRES_PASSWORD: password
          POSTGRES_DB: spatial_data_db
          POSTGRES_HOST: localhost
    steps:
      - attach_workspace:
          at: *workspace-dir
      - checkout
      - run:
          name: Install GDAL
          command: |
            apt update -y
            apt-get install libmysqlclient-dev
            apt-get install libpq-dev gdal-bin libgdal-dev -y
      - run:
          name: "Install Dependencies"
          command: |
            poetry install --no-ansi
      - run:
          name: Creating database structure
          command: |
            cd etl_app
            poetry run alembic upgrade head
            poetry run python app.py
      - run:
          name: run tests
          command: |
            cd server_app
            poetry run pytest .

workflows:
  build-and-test:
    jobs:
      - test
```

This configuration file tells CircleCI to spin up both a Python environment and a PostGIS database. It installs all dependencies, runs the ETL to populate the database, and then runs our pytest suite. This ensures that any new changes don't break existing features before they're merged.

### Conclusion
We have successfully built a multi-container application that reads data from a file, transforms and stores it in a geospatial database, and exposes it via a REST API server. This entire stack is deployable with two commands and independent of the host OS.

This series illustrates the basics of using Alembic, FastAPI, Docker, Docker Compose, and CircleCI to build a robust application. While this isn't a complete real-world example, it provides a solid foundation.

To understand the step-by-step changes, I recommend following the commit history in the public GitHub repo. I hope this two-part series was informative and useful for your own projects!

### References

All code and documented commits for the application:
[Geotechblogs official repo](https://github.com/geotechblogs/spatial_etl_app)

#### FastAPI

1. [FastAPI First Steps](https://fastapi.tiangolo.com/tutorial/first-steps/)
