---
title: "Part 2: Leaving localhost behind"
date: 2025-11-28
authors:
  - andreshernandez
_build:
  list: never
  render: always
tags:
  - Helm
  - Kubernetes
  - CircleCI
  - GCP
  - Geospatial
---

## From Code to Cluster: CI/CD and Helm for Geospatial Apps
We have built our geospatial application, defined our models, and written our tests. Now, we face the final hurdle: getting it into production reliably. In this post, we are bridging the gap between "*it works on my machine*" and "*it runs in the cloud.*"

In this post, we are focusing purely on Devops elements. We will focus on two major upgrades: setting up a robust CI/CD pipeline with CircleCI that tests against a real database, and creating a Helm chart with smart initialization logic to handle database migrations automatically upon deployment.

### 1. Real Database Testing in CI/CD
Mocking is great for unit tests, but for integration tests, nothing beats the real thing. We have updated our CircleCI configuration to spin up a live PostgreSQL service alongside our application builder.

The PostgreSQL Service

In [.circleci/config.yml](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/.circleci/config.yml), we added a PostgreSQL image to the docker executor. This ensures that when our tests run, they have a real database to talk to.

``` yaml
    docker:
      - image: cimg/python:3.12
        user: root
        environment:
          - DB_USER=user
          # ... env vars ...
      - image: postgres:15-alpine
        environment:
          - POSTGRES_USER=user
          # ... env vars ...
```

#### The "Wait" Strategy

Databases take time to boot. If your tests start immediately, they will fail. We added a crucial step to wait until the database is ready to accept connections before running the test suite.

```yaml
        - run:
          name: "Wait for database"
          command: |
            until psql -h localhost -U user -d postgres -c ";"; do
              echo "Waiting for database..."
              sleep 2
            done
```

Once the database is responsive, we run our full test suite using pytest.

### 2. Dynamic Deployments with Helm
Deploying to Kubernetes requires more than just a Docker image. We need to manage configurations, secrets, and initialization orders. We have reorganized our charts into [geoproject-charts](https://github.com/geotechblogs/geospatial_project/tree/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject-charts) and added a sophisticated Init Container pattern. But first for the newcomers to Helm world, let me introduce Helm and why it is useful on our setup.

#### What is Helm and Why Do We Need It?

Managing Kubernetes manifests manually is often called "*YAML hell.*" You end up copying and pasting static files for every environment—Development, Staging, and Production—which inevitably leads to configuration drift and broken deployments. Helm solves this by acting as a package manager for Kubernetes (like pip for Python). It allows us to define a single "blueprint" called a Chart and inject environment-specific configuration dynamically.

#### Why we chose Helm for GeoProject:
- Dependency Management: Our application isn't just a Python script; it depends strictly on a PostGIS database. Helm allows us to bundle these two together. When we run helm install, it spins up both our API and the correct version of the database automatically.
- Templating for Migrations: We need to pass the same database credentials to both our main application and our migration Init Container. Helm's templating engine lets us define these credentials once in [values.yaml](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject-charts/values.yaml) and inject them into multiple places in our manifests, eliminating the risk of typos or mismatched passwords.
- Portability: By using Helm, we ensure that the exact same code runs on our laptop (Minikube) and in the cloud (GKE). We simply swap the `values.yaml` file to change settings like NodePort to LoadBalancer without touching the core logic.

Check the [Helm Quickstart Guide](http://helm.sh/docs/intro/quickstart/) if you are interested in Helm. 

Now that we know why did we choose Helm, lets start our setup. 

#### The Init Container Pattern

We don't want our application to start until the database is up and the schema is migrated. We use a Kubernetes Init Container to handle this. It runs before the main application container starts.

In [geoproject-charts/templates/deployment.yaml](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject-charts/templates/deployment.yaml), we defined an init container that performs two critical tasks:

Waits for the Database: It uses a lightweight bash script using `/dev/tcp` to check connectivity. This is more reliable than installing netcat in slim containers.

Runs Migrations: It executes alembic upgrade head to apply schema changes automatically.

```yaml
        command: ["/bin/sh", "-c"]
          args:
            - |
              # SHELL-BASED WAIT FOR DATABASE
              echo "Waiting for PostgreSQL at $DB_HOST:$DB_PORT..."
              # ... loop logic ...
                if timeout 1 bash -c "cat < /dev/null > /dev/tcp/$DB_HOST/$DB_PORT"; then
                  echo "DB connection successful."
                  break
                fi
              # ...
              echo "Database is ready. Running Alembic migrations."
              /usr/local/bin/poetry run alembic -c geoproject/alembic/alembic.ini upgrade head
```


> [!INFO] Kubernetes Helm configuration
> Kubernetes configuration with Helm consists of mainly Charts.yaml, Values.yaml, and templates. 
> However there are other files needed, that we don't cover here.
> Please check carefully the Github Repo [geoproject-charts](https://github.com/geotechblogs/geospatial_project/tree/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject-charts) for all the info needed to deploy the app.


### 3. Dynamic Configuration for Alembic
Hardcoding database URLs in alembic.ini is a security risk and a deployment nightmare. We cleaned this up by making Alembic configuration dynamic.

First, we commented out the static `sqlalchemy.url` in [geoproject/alembic/alembic.ini](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject/alembic/alembic.ini). Then, we updated [geoproject/alembic/env.py](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject/alembic/env.py) to pull the connection string directly from our application's `get_settings()` function

```python
# geoproject/alembic/env.py
def run_migrations_online() -> None:
    settings = get_settings()
    connection_url = settings.db_url

    # Inject the URL dynamically at runtime
    config.set_main_option("sqlalchemy.url", connection_url)

    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
```

This allows us to control the database connection entirely via environment variables in [geoproject/core/config.py](https://github.com/geotechblogs/geospatial_project/blob/26dfd91fc8cac0501f1d14a214b6ba1efad958bd/geoproject/core/config.py), making the app cloud-native and 12-factor compliant.

### 4. Checking our cluster ran the app effectively
After your helm configuration is ready we can test it by running the following commands sequentially. Make sure you have set an account in docker hub and it is correctly configured. Kubernetes deployments are made directly from images stored in docker repositories (e.g. dockerhub, GKE, etc.)

```bash
docker build -t <YOUR-ACCOUNT-ID>/geoproject:latest . # building your docker image with the correct tag.
docker push <YOUR-ACCOUNT-ID>/geoproject:latest # pushing your image to the correct dockerhub account.
helm install geoproject-chart . -f values.yaml  # helm command install application from the geoproject-charts folder
```

If there is no errors, you can check the pods by running

```bash
kubectl get pods
```
and the corresponding output should be something similar to: 

```bash
NAME                                   READY   STATUS    RESTARTS   AGE
geoproject-chart-app-6746479dc-6pbdt   1/1     Running   0          2d6h
geoproject-chart-postgresql-0          1/1     Running   0          2d6h
```
Then check you app by running:
```bash
minikube service geoproject-chart-app-service --url 
```

And test it with postman or similar (same as we did in Part 1) but with the port defined by the output of the previous command.

### 5. Going Global: Deploying to Google Cloud Platform

Running helm install on Minikube is satisfying, but the real power of Kubernetes is portability. Because we have packaged everything into a Helm chart and containerized our dependencies, moving to the cloud is trivial.

Here is how you can take what we just built and deploy it to a Google Kubernetes Engine (GKE) cluster in three steps:

1. Point kubectl to the Cloud First, we need to tell our local kubectl tool to stop talking to Minikube and start talking to GKE. We do this by fetching the cluster credentials:

```bash
gcloud container clusters get-credentials <YOUR_CLUSTER_NAME> \
    --region <YOUR_REGION> \
    --project <YOUR_PROJECT_ID>
```

2. Verify the Context Ensure you are connected to the right place. You should see a GKE URI instead of "minikube".

```bash
kubectl config current-context
# Output should look like: gke_project-id_region_cluster-name
```

3. Deploy Run the exact same Helm command you used locally. The chart will handle the rest—creating the Services, Deployments, and spinning up the Init Containers to migrate the database in the cloud.

```bash
helm upgrade --install geoproject ./geoproject-charts \
    --set app.image.repository=gcr.io/<YOUR_PROJECT_ID>/geoproject \
    --set app.image.tag=latest
```


## Conclusion
We have now successfully crossed the chasm between "*it works on my machine*" and "*it runs in production.*" By replacing mock objects with live databases in our CI pipeline, we have guaranteed that our code behaves correctly under real-world conditions. By wrapping our application in Helm and utilizing Init Containers, we have created a self-healing deployment that manages its own database schema automatically.

Your application is now cloud-native, portable, and resilient. But an empty application—no matter how well-architected—is boring. We have the infrastructure; now we need the data. In Part 3, we will turn this API into a data powerhouse, implementing a serverless ingestion pipeline with DuckDB and automating complex geospatial analytics with dbt. Let's add the data engineering component.

***********************************
If you want to check the previous Blog [Part 1: The Foundation – Building a Production-Ready API]({{< ref "/blog/geospatial_data_engineering/page_1" >}}) or continue with [Part 3: The Pipeline – Serverless ETL with DuckDB and dbt]({{< ref "/blog/geospatial_data_engineering/page_3" >}})

### References

1. All code and documented commits for the application:
- [Geotechblogs geoproject official repo](https://github.com/geotechblogs/geospatial_project)
