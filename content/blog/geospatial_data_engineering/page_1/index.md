---
title: "Part 1: The Geospatial CRUD Foundation - From Script to Architecture"
date: 2025-11-28
authors:
  - andreshernandez
_build:
  list: never
  render: always
tags:
  - FastAPI
  - Docker
  - alembic
  - Geospatial
---

In the geospatial world, it is a rite of passage to write a Python script that reads a Shapefile or GeoJSON and plots it. But there is a massive leap between a script that runs on your laptop and a production API that powers a platform. Welcome to **Part 1** of our series on building a cloud-native geospatial platform. In this post, we are focusing purely on Software Engineering. We will take a raw concept and scaffold it into geoproject—a robust application built with FastAPI and PostGIS. We will move beyond requirements.txt to professional dependency management, architect a scalable Service Layer pattern to decouple our logic, and implement a testing strategy that ensures our geometry handling is bulletproof. Let's stop scripting and start engineering.

We aren't just writing code; we are building a foundation. We will look at how to structure your dependencies using Poetry, manage configuration with Pydantic, and most importantly, how to set up your database to handle complex geometry types using GeoAlchemy2 and Alembic migrations. If you are a geospatial developer looking to level up your engineering skills, this guide is for you!

## The Foundation: 
### 1. Configuration and Dependencies
Before we draw any maps, we need to ensure our environment is solid. We are moving away from requirements.txt and adopting Poetry for dependency management. This ensures that our critical geospatial libraries—like geopandas, geoalchemy2, and psycopg2—are version-controlled and compatible.

Here is a look at our pyproject.toml. Notice how we are explicitly defining our geospatial stack alongside our web framework, FastAPI:

[pyproject.toml](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/pyproject.toml)

```toml
[project]
name = "geoproject"
version = "0.1.0"
# ...
dependencies = [
    "uvicorn (>=0.38.0,<0.39.0)",
    "fastapi (>=0.121.2,<0.122.0)",
    "geoalchemy2 (>=0.18.0,<0.19.0)",
    "geopandas (>=1.1.1,<2.0.0)",
    "psycopg2 (>=2.9.11,<3.0.0)",
    "alembic (>=1.17.2,<2.0.0)",
    "pydantic-settings (>=2.12.0,<3.0.0)",
    "loguru (>=0.7.3,<0.8.0)"
]
```

To manage the application's runtime settings (like database credentials), we are using pydantic-settings. This allows us to load environment variables safely and type-check them on startup. In [geoproject/core/config.py](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/geoproject/core/config.py), we define an **ApplicationConfig** class. This centralized configuration helps avoid hardcoding sensitive strings throughout your codebase.

```python
class ApplicationConfig(BaseSettings):
    log_level: str = "INFO"
    
    # FastAPI Settings
    title: str = "geoproject"
    version: str = "0.0.1"

    # Database Settings
    db_user: str = ""
    db_password: str = ""
    db_name: str = "postgres"

    @property
    def db_url(self):
        return f"postgresql+psycopg2://{self.db_user}:{self.db_password}@localhost/{self.db_name}"
```

### 2. Defining Spatial Models
Now for the fun part: defining the data structure. Standard ORMs handle integers and strings easily, but geospatial data requires special handling. We are using GeoAlchemy2 to integrate PostGIS geometry types into our SQLAlchemy models.

In [geoproject/alembic/models/models.py](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/geoproject/alembic/models/models.py), we define a SpatialData table. Pay close attention to the geometry column. We aren't just storing text; we are storing a compiled binary geometry with a specific Spatial Reference System Identifier (SRID 4326), which represents standard latitude/longitude coordinates.

```python
import sqlalchemy as sa
from geoalchemy2 import Geometry
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.sql import func
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class SpatialData(Base):
    __tablename__ = "spatial_data"
    __table_args__ = {"schema": "public"}
    
    id = sa.Column(UUID, primary_key=True, server_default=sa.text("gen_random_uuid()"))
    timestamp = sa.Column(sa.DateTime(timezone=True), server_default=func.now())
    
    # This acts as the bridge to PostGIS
    geometry = sa.Column(
        Geometry("GEOMETRY", srid=4326, spatial_index=True), nullable=False
    )
```

### 3. Managing Database Migrations
With our model defined, we need to apply it to the database. We use Alembic to manage these schema changes. This is critical in production; you never want to be running raw CREATE TABLE scripts by hand.

We have configured Alembic to generate migration scripts that not only create the table but also set up the spatial indexes required for fast geospatial queries (like "find all points within this polygon").

Here is the migration script [c2a099e7f488_added_spatial_data_table.py](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/geoproject/alembic/versions/c2a099e7f488_added_spatial_data_table.py). Notice how it creates a GIST index on the geometry column—without this, your spatial queries would be incredibly slow:

```python
def upgrade() -> None:
    # Create the table with the geometry column
    op.create_table(
        "spatial_data",
        sa.Column("id", sa.UUID(), server_default=sa.text("gen_random_uuid()"), nullable=False),
        sa.Column("timestamp", sa.DateTime(timezone=True), server_default=sa.text("now()"), nullable=True),
        sa.Column(
            "geometry",
            geoalchemy2.types.Geometry(
                srid=4326,
                dimension=2,
                from_text="ST_GeomFromEWKT",
                name="geometry",
                nullable=False,
            ),
            nullable=False,
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="public",
    )
    # Create the spatial index for performance
    op.create_index(
        "idx_spatial_data_geometry",
        "spatial_data",
        ["geometry"],
        unique=False,
        schema="public",
        postgresql_using="gist",
    )
```

We have successfully set up the skeleton of geoproject. We have a modern configuration system, a dependency manager, and a database schema ready to ingest and index spatial data using PostGIS.


### 4. Including pre-commit hooks and gemini GitHub Actions to AI assisted coding

But we didn't stop at just the code—we also added a robust CI/CD pipeline using GitHub Actions and an initial CircleCI [configuration](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/.circleci/config.yml) along with `pre-commit` hooks to ensure our code is tested and linted automatically. Checkout the GitHub Actions workflow to enable gemini gh reviews, very useful and practical. Combining pre-commit hooks and gemini gh actions we have a very robust automatic code styling, naming conventions, and other important details on your software daily practice.

- [pre-commit.yaml](https://github.com/geotechblogs/geospatial_project/blob/7b845c492727239cc77a798dc31450a8e7280101/.pre-commit-config.yaml)
- [Github Actions Workflow](https://github.com/geotechblogs/geospatial_project/tree/7b845c492727239cc77a798dc31450a8e7280101/.github/workflows)

by running
```bash
pre-commit run --all-files
pre-commit run --files /<path-to-file-or-folder>
```

You can edit or find the errors that the linter and mypy find. Mypy helps in keeping typing consistent, and allow you to keep a code in production-ready state. The configuration options on the `pre-commit-config.yaml` will allow you to personalize and add the specific linters of your preference. GitHub Actions are enabled when you create the PR to be merged, it will trigger the actions to get very insightful code reviews that you can tune with the elements of your choice (code review example [here](https://github.com/geotechblogs/geospatial_project/pull/3#discussion_r2543855375)).


Now that the template foundation is laid, the next logical step is to build a FastAPI endpoint to accept GeoJSON payloads and save them into our new SpatialData table.

## Scaling Up: Dockerizing GDAL and Structuring a Geospatial API
In our previous session, we established the groundwork for geoproject. It is time to turn that skeleton into a fully functional, production-grade application. 

This update focuses on three critical areas: 
- Containerizing complex geospatial dependencies (GDAL), 
- Implementing a clean service-layer architecture. 
- Preparing for deployment with Helm.

We are moving beyond simple scripts to a robust structure that separates concerns: data models, business logic (services), and API routes are now distinct. This ensures your application is testable, maintainable, and ready for the cloud.

### Logic Analysis: The Service Pattern and Containerization
Before diving into the code, let's look at the architectural shift. We have introduced a Service Layer pattern. Instead of writing database queries directly inside our API routes, we delegate that logic to dedicated service functions. This keeps our routers clean and allows us to reuse logic easily.

Additionally, geospatial applications often face "dependency hell" due to binary libraries like GDAL. We solved this by hardening our Dockerfile to compile these libraries from source, ensuring our container runs anywhere without missing shared object errors.

#### Step 1: Taming the Geospatial Environment
The most significant infrastructure change is in our Dockerfile. Python packages like fiona and shapely rely on C++ libraries (GDAL/GEOS). If you have ever struggled with pip install failing on geospatial packages, this configuration is the fix.

We updated the Dockerfile to install system-level dependencies and set necessary environment variables before installing Python packages:

[Dockerfile](https://github.com/geotechblogs/geospatial_project/blob/4f710d85907ccc9fcf4674aca298fa57ee341157/Dockerfile) at this point.

```Dockerfile
# Dockerfile
RUN apt-get update && apt-get install -y \
    gdal-bin \
    libgdal-dev \
    build-essential \
    && apt-get clean

# Set environment variables for GDAL to find headers
ENV CPLUS_INCLUDE_PATH=/usr/include/gdal
ENV C_INCLUDE_PATH=/usr/include/gdal

RUN pip install fiona
```
We also added a CMD instruction to automatically launch the application with uvicorn on port 8000 when the container starts.

#### Step 2: Refining the Data Model
We renamed our generic SpatialData table to a more specific Locations table. We also switched the primary key to a UUID field named location_id and added a description field to make the data more meaningful.

[Here]((https://github.com/geotechblogs/geospatial_project/blob/4f710d85907ccc9fcf4674aca298fa57ee341157/geoproject/alembic/models/models.py):) is the updated SQLAlchemy model:

```python
# geoproject/alembic/models/models.py
class Locations(Base):
    __tablename__ = "locations"
    __table_args__ = {"schema": "public"}
    
    # Using explicit UUID type
    location_id = sa.Column(
        UUID(as_uuid=True),
        primary_key=True,
        server_default=sa.text("gen_random_uuid()"),
    )
    description = sa.Column(sa.String(255))
    timestamp = sa.Column(sa.DateTime(timezone=True), server_default=func.now())
    geometry = sa.Column(
        Geometry("GEOMETRY", srid=4326, spatial_index=True), nullable=False
    )
```

#### Step 3: Implementing the API and Services
We created a dedicated service layer in [geoproject/services/locations.py](https://github.com/geotechblogs/geospatial_project/blob/4f710d85907ccc9fcf4674aca298fa57ee341157/geoproject/services/locations.py) to handle CRUD operations. This separates the database session management from the HTTP request handling.

For example, here is the service function to create a new location. It handles the transaction commit and converts the geometry back to a shape for the response:

```python
# geoproject/services/locations.py
def create_location_service(
    location: LocationCreateUpdate, db: Session = Depends(get_session)
):
    db_location = DBLocations(
        description=location.description, geometry=location.geometry
    )
    db.add(db_location)
    db.commit()
    db.refresh(db_location)
    
    # Convert WKTElement back to a shape for the API response
    geometry = to_shape(cast(WKTElement, db_location.geometry)).__geo_interface__
    return LocationResponse(
        location_id=cast(UUID, db_location.location_id),
        timestamp=cast(datetime, db_location.timestamp),
        description=cast(str, db_location.description),
        geometry=geometry,
    )
```

We then expose this logic via a router in [geoproject/api/v1/location.py](https://github.com/geotechblogs/geospatial_project/blob/4f710d85907ccc9fcf4674aca298fa57ee341157/geoproject/api/v1/location.py). Notice how clean the endpoint function is—it simply calls the service:

```python
# geoproject/api/v1/location.py
@router.post("/", response_model=LocationResponse)
def create_location(location: LocationCreateUpdate, db: Session = Depends(get_session)):
    return create_location_service(location, db)
```

Finally, we wired this router into the main application in [geoproject/main.py](https://github.com/geotechblogs/geospatial_project/blob/4f710d85907ccc9fcf4674aca298fa57ee341157/geoproject/main.py), prefixing it with `/api/v1/locations`

We have successfully transformed a basic script into a structured, containerized application. By isolating our business logic in services and creating a robust Docker environment for GDAL, geoproject is now ready for serious development.

## Testing Your Geospatial API with Pytest and Mocking
In our previous section, we built a robust geospatial API using FastAPI, PostGIS, and Docker. We have a solid foundation, but in the software world, code that isn't tested is code that is broken. Today, we are adding the final layer of production readiness: a comprehensive test suite using pytest. While this is not by any means an effort to make robust testing, it lays the foundations to allow you to create the testing suite by configuring the minimum elements for testing the geospatial app and all its components.

#### 1. Smarter Models and Graceful Updates
Before we write tests, let's make our API a bit more user-friendly. We modified the LocationCreateUpdate model to make the geometry field optional. If a user provides a description but forgets the coordinates, we default to a standard point using default_factory.

This change prevents validation errors for simple inputs and ensures our database always gets valid spatial data

```python
# geoproject/models/locations.py
class LocationCreateUpdate(BaseModel):
    description: str
    # Geometry is now optional and defaults to (0,0)
    geometry: Optional[WKTElement] = Field(
        default_factory=lambda: WKTElement("POINT (0 0)", srid=4326)
    )
```

To support this flexibility, we also updated our service logic. In [update_location_service](https://github.com/geotechblogs/geospatial_project/blob/437959c24e1ef828589f08b08f0012fa4b9ef4ea/geoproject/services/locations.py), we now use .get() when updating fields. This ensures that if a user sends a partial update (e.g., just changing the description), we don't accidentally overwrite the existing geometry with None or crash the application

```python
# geoproject/services/locations.py
# Use .get() to preserve existing values if the key is missing in the update
db_location.description = location_data.get("description", db_location.description)
db_location.geometry = location_data.get("geometry", db_location.geometry)
```

#### 2. Setting Up the Test Infrastructure
Testing database interactions can be tricky. You rarely want your unit tests to hit a real running PostgreSQL instance—it's slow and requires complex setup. Instead, we mock the database session.

FastAPI makes this incredibly easy with Dependency Injection. We can override the get_session dependency to return a MagicMock instead of a real SQLAlchemy session. We set this up in [tests/api/v1/conftest.py](https://github.com/geotechblogs/geospatial_project/blob/437959c24e1ef828589f08b08f0012fa4b9ef4ea/tests/api/v1/conftest.py).

Here is how we configure the mock to simulate successful queries without a real database connection:

```python
# tests/api/v1/conftest.py
@pytest.fixture
def client() -> Generator[TestClient, Any, Any]:
    # Override the dependency to use our mock session
    app.dependency_overrides[get_session] = override_get_session
    yield TestClient(app)
    # Clean up after tests
    app.dependency_overrides.clear()

@pytest.fixture
def configured_mock_get_all_locations():
    def mock_all_locations():
        return [
            DBLocations(
                location_id=uuid.uuid4(),
                timestamp=datetime.now(timezone.utc),
                description="Test Location",
                geometry=WKTElement("POINT (102.0 0.5)", srid=4326),
            )
        ]
    MOCK_DB_SESSION.reset_mock()
    MOCK_DB_SESSION.query.return_value.all.return_value = mock_all_locations()
```

#### 3. Writing the Unit Tests
With our fixtures in place, writing the actual tests is straightforward. We create a new file [tests/api/v1/test_locations.py](https://github.com/geotechblogs/geospatial_project/blob/437959c24e1ef828589f08b08f0012fa4b9ef4ea/tests/api/v1/test_locations.py) to cover our CRUD operations.

We use pytest.mark.parametrize to test different inputs efficiently. Notice how we assert both the status code and the JSON structure to ensure our serialization logic (converting WKT back to GeoJSON) is working correctly.

```python
# tests/api/v1/test_locations.py
def test_create_location_should_return_correct_location(
    client: TestClient,
    configured_mock_session: Generator[None, None, None],
    location_data: Dict[str, Any],
):
    response = client.post("/api/v1/locations", json=location_data)

    assert response.status_code == 200
    assert response.json()["description"] == location_data["description"]
    # Verify the GeoJSON response structure
    assert response.json()["geometry"] == location_data["geometry"]
```

We also added a test case to verify our new default geometry logic. If we send a payload with only a description, the API should automatically assign it to coordinate (0,0)

```python
def test_create_location_should_return_default_geometry(client: TestClient):
    response = client.post("/api/v1/locations", json={"description": "Test Location"})
    assert response.status_code == 200
    # Confirm the default factory worked
    assert response.json()["geometry"] == {"type": "Point", "coordinates": [0.0, 0.0]}
```

We have now successfully mocked our database layer and written comprehensive unit tests for our geospatial API. This setup allows us to refactor with confidence, knowing that any regressions in our API contract or logic will be caught immediately.

#### 4. Testing our app locally
After adding the geospatial CRUD and the testing, we can also tests locally our app, by using our tool of preference e.g. postman after running the app. For debugging purposes, I included the launch.json here file to run the app in VSCode or forks IDEs (e.g. Antigravity, or Cursor)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python Debugger: FastAPI",
            "type": "debugpy",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "geoproject.main:app",
                "--reload"
            ],
            "jinja": true
        },
    ]
}
```

with Postman the payload

```json
{
  "description": "Boundaries of the Company's regional sales territory.",
  "geometry": {
        "type": "Polygon",
        "coordinates": [ [ [-122.41, 37.77], [-122.45, 37.77], [-122.45, 37.8], [-122.41, 37.8], [-122.41, 37.77]]]
    }
}
```
to the endpoint [http://127.0.0.1:8000/api/v1/locations]()

Will produce the same response, after saving the data to the database:

| location_id | Description | TImestamp | Geometry |
| :--- | :--- | :--- | :--- |
| c8c34a02-a5d1-4213-889d-5d0889b6d86e | Boundaries of the Company's regional sales territory | 2025-11-20 19:59:12.635 +0100 | POLYGON ((-122.41 37.77, -122.45 37.77, -122.45 37.8, -122.41 37.8, -122.41 37.77)) |

```json
{
    "location_id": "c8c34a02-a5d1-4213-889d-5d0889b6d86e",
    "timestamp": "2025-11-20 19:59:12.635582Z",
    "description": "Boundaries of the Company's regional sales territory.",
    "geometry": {
        "type": "Polygon",
        "coordinates": [ [ [-122.41, 37.77], [-122.45, 37.77], [-122.45, 37.8], [-122.41, 37.8], [-122.41, 37.77]]]
    }
}
```

## Conclusion

We have successfully elevated a simple concept into a professional-grade backend. We now have a clean architecture where our business logic is isolated in a Service Layer, our database schema is version-controlled with Alembic, and our complex dependencies (like GDAL) are tamed within a Docker container. Most importantly, we have established a testing culture with Pytest, allowing us to refactor with confidence.

You now have a solid, testable codebase, but code living on a laptop is not a product. We have built the engine, but we still need to build the car around it. In Part 2, we will leave localhost behind. We will take this container, harden it for production, and orchestrate it in the cloud using Helm and Kubernetes.

## References
- Blog series [Building a Cloud-Native Geospatial Platform: From Localhost to Kubernetes]({{< ref "/blog/geospatial_data_engineering/main_page" >}})
- Continue with [Part 2: The Infrastructure – Dockerizing GDAL and Deploying with Helm]({{< ref "/blog/geospatial_data_engineering/page_2" >}})

All code and documented commits for the application:
- [Geotechblogs geoproject official repo](https://github.com/geotechblogs/geospatial_project)