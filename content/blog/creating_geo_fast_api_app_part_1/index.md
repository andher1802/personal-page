---
title: 📈 Building a Self-Served Geospatial ETL Pipeline Part 1 (PostGIS, Docker & Alembic)
summary: .
date: 2025-11-07
authors:
  - andreshernandez
tags:
  - Statistics
  - Logistic Regression
image:
  caption: 'Logistic Regression'
---

### The Challenge: Beyond a Simple Script

Loading a CSV file into a database might sound simple. But what happens when you need to handle complex spatial data types, manage database schema changes gracefully, and deploy the entire process reliably in a portable, OS-independent package? This is where a simple task becomes a complex engineering challenge.

Modern geospatial data science demands more than just one-off scripts. We need automated, scalable, and deployable solutions. In this two-part series, I'll walk you through building a complete, self-served geospatial application from scratch.

This first post focuses on the foundation: creating a robust, multi-container ETL service. We will use Docker and docker-compose to orchestrate our services, PostGIS as our spatially-aware database, and Alembic to manage our database schema with version control.

By the end of this post, you will have a fully containerized service that, with a single command, builds a database, applies all schema migrations, and loads spatial data.

### The Vision: A Multi-Container Geospatial App
Our goal is to build an application that not only performs an ETL (Extract, Transform, Load) process but is also scalable, manageable, and ready for production.

Here’s the design:

The ETL Service: I chose a Python script using SQLAlchemy to load data into a PostGIS database. The real power move here is adding Alembic, a database migration tool. It allows us to manage changes to our database schema in a controlled, versioned, and repeatable way.

The REST API Server: The end goal (which we'll tackle in Part 2) is to implement a FastAPI server. I chose it because of its gentle learning curve and robust, high-performance nature, making it perfect for exposing our data in a RESTful manner.

Continuous Integration (CI): The entire system is designed with CI in mind. We will configure a workflow (in CircleCI) to run pytest on every push, ensuring code quality. Ideally, this includes protecting the main branch so merges are only allowed via pull requests that pass all tests.

Docker & Containerization: The key to making this "self-served" is containerization. Our app has three main components (database, ETL, API), and Docker allows us to package and run them anywhere. We'll use docker-compose to define and run this multi-container application.

```bash
echo "# spatial_etl_app" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:geotechblogs/spatial_etl_app.git
git remote set-url origin git@github.com:geotechblogs/spatial_etl_app.git
git push -u origin main
```

```bash
poetry init # Follow the steps in the interactive cmd window
mkdir etl_app server_app
```

### Implementation: The Database Foundation
Let's dive into the build, starting with the core: the PostGIS database container.

We'll use docker-compose.yml to define our services. The first service is the database itself, using the official postgis/postgis image. This image gives us a Postgres database fully optimized for GIS operations.

```bash
version: '3'
services:
  database:
    image: postgis/postgis:13-3.1
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
      POSTGRES_DB: spatial_data_db
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

After defining the service in the file and running the Docker commands, you will have a running PostGIS server. You can immediately connect to the spatial_data_db on port 5432 using your preferred database client (like DBeaver) to verify it's working.

```bash
brew install docker-compose # for Mac to install the docker-compose command
docker-compose build
docker-compose up -d
```

### Implementation: Managing Schema with Alembic
A running database is great, but we should never make schema changes manually in a production-style app. For this, we use Alembic.

First, we add our Python dependencies, including alembic, geoalchemy2 (for spatial data types), and psycopg2. We then initialize Alembic, which creates our migration environment.

```bash
poetry add geoalchemy2 geopandas psycopg2 alembic
cd etl_app
alembic init .
```

The key step is configuring Alembic. We edit the alembic.ini file to point to our database (for now, localhost is fine for local testing). 

```bash
sqlalchemy.url = postgresql+psycopg2://username:password@localhost:5432/spatial_data_db # sqlalchemy.url = driver://user:pass@localhost/dbname
```

We also define our table structure in a declarative Python model. For this app, it’s a simple table with an ID, a timestamp, and a geometry column.

```python
import sqlalchemy as sa
from geoalchemy2 import Geometry
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.sql import func

Base = declarative_base()


class SpatialData(Base):
    __tablename__ = "spatial_data"
    __table_args__ = {"schema": "public"} # you can set your custom schema but make sure it exists in the database
    
    id = sa.Column(UUID, primary_key=True, server_default=sa.text("gen_random_uuid()"))
    timestamp = sa.Column(sa.DateTime(timezone=True), server_default=func.now())
    geometry = sa.Column(
        Geometry("GEOMETRY", srid=4326, spatial_index=True), nullable=False
    )
```

Once the model is defined, we tell Alembic to find it. 

```python
from models.models import Base
...
target_metadata = Base.metadata
```

Then, we run the alembic revision --autogenerate command. 

```bash
alembic revision --autogenerate -m "Added spatial_data table"
```

Alembic inspects our model, compares it to the database, and automatically generates a new Python migration script.

```python
"""Added spatial_data table

Revision ID: 1b9591aa6287
Revises: 
Create Date: 2023-12-26 18:17:01.902266

"""
from typing import Sequence, Union

import geoalchemy2
import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision: str = '1b9591aa6287'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto-generated by Alembic - please adjust! ###
    op.create_table('spatial_data',
    sa.Column('id', sa.UUID(), server_default=sa.text('gen_random_uuid()'), nullable=False),
    sa.Column('timestamp', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=True),
    sa.Column('geometry', geoalchemy2.types.Geometry(srid=4326, from_text='ST_GeomFromEWKT', name='geometry', nullable=False), nullable=False),
    sa.PrimaryKeyConstraint('id'),
    schema='public'
    )
    # ### end Alembic commands ###


def downgrade() -> None:
    pass
```

Finally, we run alembic upgrade head. This command applies the new script, and voila! Our spatial_data table, complete with its geometry column, is created in the database. We now have a version-controlled, repeatable way to manage our database structure.

```bash
alembic upgrade head
```

### Implementation: The Python ETL Script
With a version-controlled database, we can now write the ETL script. This script is responsible for three steps:

- Extract: Read data from our fakedata.csv file.

```bash
org_id;timestamp;latitude;longitude
1;2023-02-05 19:40:51;-56.572095;-172.067458
2;2023-08-13 18:04:18;-76.839549;-155.767195
3;2023-07-17 16:30:07;57.937777;-98.577786
4;2023-04-16 04:42:38;-2.411921;-139.246176
5;2023-03-17 08:13:46;80.271008;-11.893289
6;2023-04-25 10:59:03;60.16493;-59.32429
7;2023-06-14 08:29:56;-36.82034;156.306733
8;2023-08-21 13:23:17;-64.846411;42.986379
9;2023-04-25 06:08:50;-5.19421;-103.131961
10;2023-04-30 10:14:37;-82.534738;-89.653191
11;2023-05-09 02:22:23;34.534001;-20.195419
12;2023-05-15 10:52:20;59.900658;61.658177
13;2023-07-01 18:31:36;50.275187;10.261205
14;2023-04-15 03:04:39;-89.575288;-49.960057
15;2023-01-31 03:04:31;-76.67953;59.677759
16;2023-06-11 01:25:09;79.355176;132.943352
17;2023-04-25 01:24:20;63.466903;64.39265
18;2023-09-14 16:47:03;36.617167;-56.158655
19;2023-02-18 19:06:39;-64.280607;98.59213
```

- Transform: Use GeoPandas to convert the latitude and longitude columns into a GeoDataFrame with a proper geometry object.

- Load: Use SQLAlchemy to create a database engine and write the GeoDataFrame directly into our PostGIS table.

```python
import geopandas as gpd
import pandas as pd
from sqlalchemy import create_engine

INPUT_PATH = "./data/fakedata.csv"
df = pd.read_csv(INPUT_PATH, sep=";")
gdf = gpd.GeoDataFrame(
    df[["org_id", "timestamp"]],
    geometry=gpd.points_from_xy(df["longitude"], df["latitude"]),
    crs="EPSG:4326",
)
# Create a database connection
db_engine = create_engine(
    "postgresql://username:password@localhost:5432/spatial_data_db"
)

gdf.to_postgis("spatial_data", db_engine, if_exists="append", index=False)
db_engine.dispose()
```

During this process, I hit a common snag: my initial schema used a UUID for the ID, but my fake data used a simple integer. This is a perfect use case for Alembic! Instead of manually altering the database, I simply changed the model in my Python code,

``` python
# remove this line 
# id = sa.Column(UUID, primary_key=True, server_default=sa.text("gen_random_uuid()"))
# and add the following field
org_id = sa.Column(sa.Integer, primary_key=True)
```

ran alembic revision --autogenerate to create a new migration script, and ran alembic upgrade head.

``` python
"""Adding org_id to the spatial_data table

Revision ID: 8261e3378fff
Revises: 1b9591aa6287
Create Date: 2023-12-26 19:42:56.197628

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision: str = '8261e3378fff'
down_revision: Union[str, None] = '1b9591aa6287'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('spatial_data', sa.Column('org_id', sa.Integer(), nullable=False))
    op.drop_column('spatial_data', 'id')
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('spatial_data', sa.Column('id', sa.UUID(), server_default=sa.text('gen_random_uuid()'), autoincrement=False, nullable=False))
    op.drop_column('spatial_data', 'org_id')
    # ### end Alembic commands ###
```

```bash
python app.py
```

The database schema was updated instantly, with no manual SQL required.

After running the main app.py script, our fake data was successfully loaded into the PostGIS database.

### Tying It All Together: The Multi-Container App
We've tested each piece locally, but the real goal is automation. The final step for Part 1 is to make this entire process—database creation, migration, and data loading—run from a single docker-compose up command.

This requires a few powerful additions to our docker-compose.yml:

```yml
version: '3'
services:
  database:
    image: postgis/postgis:13-3.1
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
      POSTGRES_DB: spatial_data_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - mynetwork
    healthcheck:
      test: "exit 0"

  alembic:
    build:  
      context: ./
      dockerfile: ./etl_app/Dockerfile
    environment:
      DATABASE_URL: postgresql://username:password@database:5432/spatial_data_db
    command: ["bash", "-c", "alembic upgrade head && python app.py"]
    depends_on:
      database:
        condition: service_healthy
    networks:
      - mynetwork

volumes:
  postgres_data:

networks:
  mynetwork:
    driver: bridge
```

A new alembic service: This service is built from a Dockerfile that includes Python and all our dependencies (like GeoPandas and Alembic).

```dockerfile
FROM python:3.10-slim

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

COPY poetry.lock ./
COPY pyproject.toml ./
```

A shared network: We create a mynetwork so the alembic container can find the database container by its service name.

Service Orchestration: We use depends_on and healthcheck to ensure the alembic service only runs after the database is fully healthy and ready for connections.

The "One-Two Punch": The command for our alembic service is a simple shell command: first, it runs alembic upgrade head to ensure the database schema is at the latest version, and then it runs python app.py to load the data.


This orchestration is important to avoid non-deterministic behavior.

To visualize how the database and alembic services interact, here's a diagram of the complete workflow that's orchestrated by our docker-compose.yml file

```mermaid
graph TD
    A[User] -- 1. Runs --> B[docker-compose build & up]

    subgraph "Docker Environment (Shared Network)"
        direction TB
        
        B -- 2. Starts --> C["'database' Service (PostGIS)"]
        C -- 3. Reports Healthy --> B
        
        B -- 4. Starts --> D["'alembic' Service (Python)"]
        
        subgraph "'alembic' Container Logic"
            direction TB
            D --> E[Command 1: 'alembic upgrade head']
            E --> F[Command 2: 'python app.py']
        end

        E -- 5. Connects & Migrates Schema --> C
        
        subgraph "ETL Process (app.py)"
            direction TB
            F -- 6. Reads --> G[fakedata.csv]
            G --> H[Transform with GeoPandas]
            H -- 7. Connects & Writes GeoData --> C
        end
    end
```
The final tweak is to update our Python scripts and Alembic configuration to read the database address from a DATABASE_URL environment variable, which we pass in via the docker-compose.yml file.

[etl_app/app.py](https://github.com/geotechblogs/spatial_etl_app/commit/7ecfc5f10f255221bd20b845741cff1492768933#diff-6fabdd4cfc9bde1eaf6c5b5c3af3fa3e46fb74b1d8e54689221dde8a5a6db740)

```python
import os
...

DATABASE_URL = os.getenv("DATABASE_URL")
...
# db_engine = create_engine("postgresql://username:password@localhost:5432/spatial_data_db")
db_engine = create_engine(DATABASE_URL) # Remove the above line and add this line
...
```

[etl_app/alembic.ini](https://github.com/geotechblogs/spatial_etl_app#diff-711e1c9a0344c436be5a082dc4e5ea50ffa2beea423328374ab3cf312c9262d5)

```python
...
#sqlalchemy.url = postgresql+psycopg2://username:password@database:5432/spatial_data_db
# Comment this line to make the sqlalchemy.url be read from the env.py configuration
...
```

[etl_app/env.py](https://github.com/geotechblogs/spatial_etl_app/commit/7ecfc5f10f255221bd20b845741cff1492768933#diff-2c5c75054fc90014906b797b73a05c09ae01a81cddede767c411d96fffb33bbf)
```python
import os
...
config = context.config
# Add the following line after the config variable definition to add the sqlalchemy.url read from the env var
config.set_section_option(
    "alembic",
    "sqlalchemy.url",
    os.environ.get(
        "DATABASE_URL", config.get_section_option("alembic", "sqlalchemy.url")
    ),
)
...
```

Now, after removing the old local data, we can run docker-compose build and docker-compose up. The result? The exact same tables and data are created, but this time, the entire process is fully automated and orchestrated within a multi-container environment.

``` bash
docker-compose build
docker-compose up -d
```

### Part 1 Complete: A Solid Foundation
And with that, we have a solid, deployable foundation. By simply running docker-compose build and docker-compose up, anyone can launch a fully configured PostGIS database, have the schema automatically migrated to the latest version, and load it with data. This is the power of a container-based, self-served application.

We've built a powerful ETL, but the data is still locked in the database. In Part 2, we'll complete the vision: we will build a high-performance FastAPI server to expose this data via a REST API and implement a full CI pipeline with CircleCI to automate testing and integration.

This project is part of my work exploring robust, developer-friendly spatial data systems. I hope this guide is useful. Your feedback is invaluable as I navigate through these exciting topics!

### References

All code and documented commits for the application:
[Geotechblogs official repo](https://github.com/geotechblogs/spatial_etl_app)

#### GitHub Configuration

1. [Generate new ssh-keys in Mac](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
2. [Add SSH keys to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
3. [Managing multiple GitHub accounts on your Mac](https://www.haccks.com/posts/github-multi-ssh-key/)

##### Alembic

1. [Alembic tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
