# PEOPLE-ECCO

> **Enhancing Ecosystems Conservation through Earth Observation Solutions, Capacity Development and Co-design**

PEOPLE-ECCO is an initiative funded by the **European Space Agency (ESA)** dedicated to co-designing Earth Observation (EO) methodologies, platform architectures, and software interfaces alongside Civil Society Organizations (CSOs) and Non-Governmental Organizations (NGOs) active in global ecosystem conservation.

Our primary technical focus is removing the operational friction of utilizing raw remote sensing data. We achieve this by providing a modern, open-source software platform where conservation groups can discover, configure, track, and execute cloud-based Earth Observation algorithms seamlessly.

---

## Architecture Overview

The PEOPLE-ECCO platform is decoupled into specialized presentation layers, a centralized API orchestration backend, and distributed computation runtimes:

```text
                  ┌───────────────────────────────────────────────┐
                  │          PEOPLE-ECCO Platform Frontend        │
                  └───────────────────────┬───────────────────────┘
                                          │
                                     [ ecco-web ]
                            ┌─────────────▼─────────────┐
                            │   Web Portal Dashboards   │
                            └─────────────┬─────────────┘
                                          │
                            ┌─────────────▼─────────────┐
                            │         ecco-api          │
                            │ (Job & Execution Manager) │
                            └─────────────┬─────────────┘
                                          │ [ Prefect deployment ]
                            ┌─────────────▼─────────────┐
                            │ Docker Algorithm Container│
                            │  based on ecco-algorithm- │
                            │  base + Algorithm.run(...)│
                            └──────┬──────────────┬─────┘
                                   │              │
                         [ openEO / CDSE ]   [ STAC + assets ]
                                   │              │
              ┌────────────────────▼──────┐       ▼
              │    openEO backend / UDF   │   S3-compatible
              │   (Distributed Executions)│   object storage
              └───────────────────────────┘
```

## PEOPLE-ECCO Solutions

The project develops six thematic solutions for EO data analysis. These can be categorized into three topics:

1. Marine application
2. Terrestrial application
3. Counterfactual assements (Before-After control impact)

### Solution components

#### Marine solutions

The arine solutions feature two algoriths:

1. Submerged Aquatic Vegetation (SAV)
2. Marine Habitat Frequency and Connectivity

The implementation is available at [`ecco-marine-solutions`](https://github.com/PEOPLE-ECCO/ecco-marine-solutions).

#### Terrestrial solutions

The terrestrial solutions feature two algoriths:

1. Vegetation Productivity Trend (VPT)
2. Disturbance Occurrence 
3. Habitat Disturbance

The implementation is available at [`ecco-terrestrial-solutions`](https://github.com/PEOPLE-ECCO/ecco-terrestrial-solutions).

#### Before-After control impact solution

The BACI implementation supports users with statistical features for evaluting the impact of interventions at a given site.
The implementation is available at [`ecco-baci-solutions`](https://github.com/PEOPLE-ECCO/ecco-baci-solutions).

## Platform components

### Web portal

[`ecco-web`](https://github.com/PEOPLE-ECCO/ecco-web) is a Open Pioneer Trails web application for browsing and visualizing PEOPLE-ECCO geospatial scenario data. It presents scenario sites, their time series, processing jobs, and generated spatial results through a map-centered interface (based on OpenLayers).

`ecco-web` acts as a geospatial exploration frontend for scenario-based environmental data. It keeps the backend data model hierarchical and lazy-loaded, while presenting it to users as an interactive workflow: choose a site, inspect or create time series, browse processing jobs and metrics, and visualize raster or vector results directly on the map.


#### API Interaction

The app communicates with the `ecco-api` backend through Open Pioneer’s `HttpService`.

Authentication is handled via the OIDC protocol through Keycloak.


#### Data Organization

The frontend organizes  data as a hierarchy:

```text
Scenario / Site
  -> Timeseries
      -> Job(s)
          -> Result(s)
          -> Logs
```

#### User Presentation

The main user-facing views are:

##### Sites View

The sites view loads all scenarios and presents them as cards with image, name, and description. A map beside the cards shows each site’s bounding box. Hovering over a site highlights its spatial extent, and clicking the card opens the site detail page.

##### Site Details View

The site detail view is centered around a large OpenLayers map and a left-side data panel. The map initially zooms to the selected site’s bounding box.

The data panel lists the site’s time series. Selecting a time series:

- zooms the map to the time series extent
- draws the time series bounding box
- loads its jobs on demand
- exposes available result metrics

##### Metrics and Results

Results are grouped by result type, for example:

- algorithm specific raster outputs (e.g. predictions, classifications and probabilities)
- algorithm specific vector outputs (e.g. habitat zones)
- Sentinel-2 pseudocolor/raw files for reference

Each metric group shows its temporal range. Expanding or selecting a metric renders the corresponding result layer on the map. Users can navigate through the results via a time slider, adjust layer opacity and download the currently selected result.

Raster results are displayed as GeoTIFF/WebGL tile layers. Vector results are displayed as GeoJSON layers. Some vector layers support feature click interactions that highlight related habitat patches and show popup metrics such as area and connected habitat area.

##### Time Series Creation

Users can create a new time series through a stepped dialog:

1. Enter name and description.
2. Select an available process from the API.
3. Draw an area of interest on a map.
4. Review the entered data and submit it.

The selected extent is validated before submission, and the resulting time series is added back into the UI.

### Backend API

[`ecco-api`](https://github.com/PEOPLE-ECCO/ecco-api) is an asynchronous Python API for the PEOPLE-ECCO platform. It exposes scenario, process, timeseries, job, STAC, upload, and user endpoints; stores platform metadata in PostgreSQL/PostGIS; uses S3-compatible object storage for raster/STAC artifacts; and delegates Earth observation processing to Prefect deployments that run openEO-based algorithms.

#### High-Level Architecture

```text
Client / UI
  |
  | HTTP API
  v
Quart app (`app.py`)
  |
  |-- Auth middleware (`auth/auth.py`)
  |-- STAC blueprint (`stac/routes.py`)
  |-- SQLAlchemy sessions (`storage/db.py`)
  |-- Garage/S3 client
  |-- Prefect orchestration client
  |
  | metadata                     | artifacts/results             | compute
  v                              v                               v
PostgreSQL + PostGIS          S3-compatible storage           Prefect + Docker + openEO/CDSE
(`storage/definitions.py`)    (Garage API)        (`eo/algorithms/*`)
```

The API process is served by Hypercorn with Quart and CORS enabled. It is designed to run behind a proxy, using Hypercorn's `ProxyFixMiddleware`, and can be containerized with the included `Dockerfile` and `docker-compose.yml`.

#### Core Components

##### Database and Authorization Model

`storage/definitions.py` defines the SQLAlchemy ORM model:

- `User`: platform user mapped to a Keycloak UUID.
- `Scenario`: predefined study area or case study with bounding box, description, preview image, and ACL.
- `Process`: executable algorithm available for a scenario, linked to a Prefect deployment.
- `Timeseries`: user-created spatial selection and process binding within a scenario.
- `Job`: one execution of a timeseries/process combination, linked to a Prefect flow run.

##### Object Storage and STAC

S3-compatible storage is accessed through the Garage Python client. The API stores and retrieves:

- Uploaded source data.
- STAC JSON documents.
- Algorithm output assets.
- Per-job `stac.collection` files.

`stac/routes.py` exposes STAC-related endpoints:

- `GET /stac/<path:item>` fetches STAC JSON from the `ecco` bucket and rewrites asset links to short-lived presigned URLs.
- `GET /stac/data/<path:path>` redirects to a presigned object URL.
- `POST /stac/upload` returns a presigned PUT URL for direct client uploads.
- `POST /stac/upload_success` schedules background STAC generation after an upload notification.

`stac/util.py` contains upload URL generation, object fetching, and a placeholder `generate_stac` function for future STAC item/catalog generation.

##### Processing and Job Orchestration

Processes are represented in the database by `Process` rows. Each process points to a Prefect deployment ID/name. When a client creates a job:

1. `POST /timeseries/<timeseries_id>/jobs/` loads the timeseries and its process.
2. The API merges request parameters with the timeseries geometry as `spatial_extent`.
3. Prefect `run_deployment` starts the configured deployment asynchronously.
4. A `Job` row is created with the Prefect flow run name and UUID.
5. Later job status and logs are read from Prefect.
6. Results are read from the S3 bucket named after the Prefect flow run.

`eo/algorithms/prefect_wrapper.py` defines the Prefect flow used by algorithm deployments. It:

- Loads an algorithm module dynamically from `ALGORITHM_BASE`.
- Authenticates to the Copernicus Data Space openEO federation using Prefect `Secret` blocks.
- Tracks openEO batch jobs created during algorithm execution.
- Persists generated assets to S3.
- Writes a STAC item collection as `stac.collection`.
- Pulls openEO logs/costs and stores them in Prefect logs/tags.

`eo/algorithms/cwl_wrapper.py` provides a similar dynamic algorithm runner for local/CWL-style execution. It reads parameters from a file, runs an algorithm, copies outputs into a local output directory, writes `collection.json`, and persists openEO logs.


#### API Features

##### Documentation

- `GET /`: serves ReDoc.
- `GET /openapi.yaml`: serves the OpenAPI document.

##### Scenarios and Processes

- `GET /scenarios/`: list readable scenarios.
- `GET /scenarios/<scenario_id>`: fetch one scenario.
- `GET /scenarios/<scenario_id>/processes/`: list processes available for a scenario.

##### Timeseries

- `GET /scenarios/<scenario_id>/timeseries/`: list timeseries for a scenario.
- `POST /scenarios/<scenario_id>/timeseries/`: create a timeseries by binding a name, description, process, bounding box, and GeoJSON geometry to a scenario.
- `GET /timeseries/<timeseries_id>/jobs/`: list jobs for a timeseries.
- `GET /timeseries/<timeseries_id>/download`: stream a ZIP archive containing all available job assets and STAC collections for a timeseries.

##### Jobs

- `POST /timeseries/<timeseries_id>/jobs/`: start a new Prefect/openEO execution for a timeseries.
- `GET /jobs/<job_id>/`: fetch a job merged with Prefect flow run metadata.
- `GET /jobs/<job_id>/log/`: fetch Prefect logs for the job's flow run.
- `GET /jobs/<job_id>/results`: fetch the job STAC collection with asset links rewritten as presigned S3 URLs.

##### User

- `GET /user/`: return the current user or create it from request-scope identity data.

##### STAC and Uploads

- Browse STAC JSON stored in object storage.
- Rewrite STAC asset links to presigned URLs.
- Generate presigned upload URLs for direct client uploads.
- Accept upload success notifications for background STAC generation.

#### Data Flow Examples

##### Creating and Running a Timeseries

```text
Client selects scenario + process + geometry
  -> POST /scenarios/<scenario_id>/timeseries/
  -> Timeseries row stored in PostgreSQL with inherited scenario ACL
  -> Client submits execution parameters
  -> POST /timeseries/<timeseries_id>/jobs/
  -> Prefect deployment starts
  -> Job row stores flow_run_id and flow_run_name
```

##### Persisting Results

```text
Prefect flow runs algorithm
  -> Algorithm creates openEO jobs/assets
  -> Wrapper builds a STAC ItemCollection
  -> Assets and `stac.collection` are uploaded to S3 bucket named after flow run
  -> API reads the collection for /jobs/<job_id>/results
  -> API signs asset URLs before returning JSON
```

##### Downloading a Timeseries Archive

```text
GET /timeseries/<timeseries_id>/download
  -> API streams a ZIP response
  -> Writes README.txt first
  -> Iterates all jobs for the timeseries
  -> Reads each job's `stac.collection`
  -> Downloads each referenced asset from S3
  -> Adds assets and collection.json files to the ZIP
```

#### Runtime and Deployment

The API container uses:

- Python 3.13 Alpine.
- GDAL and build dependencies.
- Hypercorn configured in `hypercorn.conf.py`.
- Port `5000`.
- One `uvloop` worker.

`docker-compose.yml` defines local services for:

- `api`: the Quart/Hypercorn service.
- `minio`: S3-compatible object storage.
- `postgres`: PostgreSQL 17 database exposed on host port `5431`.

Algorithm containers use the [`ecco-algorithm-base`](https://github.com/PEOPLE-ECCO/ecco-algorithm-base) Dockerfiles. The standard image is based on Python 3.12 Bookworm with `uv`, GDAL/PROJ dependencies, the Prefect wrapper, and the CWL wrapper; `plain.Dockerfile` provides a slimmer variant for algorithms that do not need the full geospatial system dependency set.

### Algorithm implementations

PEOPLE-ECCO algorithms are implemented as small Python modules that plug into the shared execution contract defined by [`ecco-algorithm-base`](https://github.com/PEOPLE-ECCO/ecco-algorithm-base). Each algorithm image supplies the domain-specific code and dependencies, while the base image supplies the common runtime behavior: openEO authentication, batch job tracking, Prefect integration, STAC result handling, and persistence to object storage.

An algorithm module exposes an `Algorithm` class with a static `run` method:

```python
class Algorithm:
    @staticmethod
    def run(conn: Connection, catalog: Catalog, parameters: Dict) -> None:
        ...
```

The module is selected at runtime through the `ALGORITHM_BASE` environment variable. The wrapper imports this module dynamically, creates a pre-authenticated openEO connection, creates an empty PySTAC catalog for the run, and calls `Algorithm.run(...)` with:

- `conn`: an authenticated openEO connection to the Copernicus Data Space openEO federation.
- `catalog`: a PySTAC catalog that the algorithm populates with result items and assets.
- `parameters`: the execution parameters submitted through the API, including the spatial extent derived from the selected timeseries.

The algorithm itself is responsible for the scientific workflow: loading EO collections, building openEO process graphs, starting batch jobs, downloading or generating result assets, and adding those assets to the provided STAC catalog. The shared wrapper intercepts `conn.create_job(...)` so openEO batch jobs can be tracked, logged, and costed after execution.

#### Prefect execution

For platform execution, `prefect_wrapper.py` defines the `run_algorithm` Prefect flow used by algorithm deployments. It:

- Reads CDSE and S3 credentials from Prefect `Secret` blocks.
- Authenticates to openEO using OIDC client credentials.
- Runs the selected `Algorithm` implementation.
- Uploads every asset referenced by the STAC catalog to the S3-compatible bucket named after the Prefect flow run.
- Stores the generated item collection as `stac.collection`.
- Persists openEO logs and total reported costs back into Prefect logs/tags.

This keeps algorithm code focused on EO logic and result metadata, while deployment-level concerns remain centralized in the base wrapper.

#### Local/CWL execution

`cwl_wrapper.py` provides the same algorithm contract for local or CWL-style execution. It reads:

- `ALGORITHM_BASE`: Python module containing the `Algorithm` class.
- `PARAMETERS_FILE`: JSON input parameters.
- `OUTPUT_DIR`: directory where outputs are written.
- `RUN_NAME`: optional run identifier.
- `CDSE_CLIENT_ID` and `CDSE_CLIENT_SECRET`: openEO credentials.

Instead of uploading to S3, the CWL wrapper copies referenced assets into `OUTPUT_DIR`, writes a `collection.json` STAC item collection, and writes `openeo_logs.txt` with tracked job logs, descriptions, and total costs.

#### Earth Observation Application Package / APEx compatibility

The algorithm base keeps PEOPLE-ECCO algorithms compatible with Earth Observation Application Package (EOAP) and APEx-style execution by exposing the same algorithm contract through a CWL `CommandLineTool`. The CWL layer does not need to know the algorithm internals; it only needs to run the shared `cwl_wrapper.py` inside the algorithm container and provide the expected environment variables.

The [`eoap-example`](https://github.com/PEOPLE-ECCO/eoap-example) repository shows this pattern in a minimal package. Its `run.cwl` declares a workflow with JSON `parameters`, CDSE credentials, and a `run_name`, then uses a `DockerRequirement` to run the algorithm image. The CWL environment maps those inputs to `ALGORITHM_BASE`, `PARAMETERS_FILE`, `CDSE_CLIENT_ID`, `CDSE_CLIENT_SECRET`, `RUN_NAME`, and `OUTPUT_DIR`, and invokes:

```sh
python -u /app/cwl_wrapper.py
```

Because `cwl_wrapper.py` writes all assets, `collection.json`, and `openeo_logs.txt` into the declared output directory, the same algorithm image can run both as a PEOPLE-ECCO Prefect deployment and as a portable EOAP/APEx package.

#### Adding a new algorithm

To add a new algorithm:

1. Create a Python package containing `Algorithm.run(conn, catalog, parameters)`.
2. Use `conn` for openEO data access and batch execution.
3. Write result files locally during execution.
4. Add each result to the provided PySTAC `catalog` with assets pointing to the generated files.
5. Build an algorithm-specific container from `ecco-algorithm-base`, install the algorithm's Python dependencies, copy the package into the image, and set `ALGORITHM_BASE` to the package/module name.
6. Register the resulting image as a Prefect deployment and link that deployment from the API `Process` row.

At runtime, the frontend selects a scenario process, the API starts the linked Prefect deployment, the base wrapper runs the algorithm, and the API later reads the persisted STAC collection to expose map layers and downloads.


# Funding

PEOPLE-ECCO is funded by the [European Space Agency](https://www.esa.int/) as one of the projects in the Pioneering Earth Observation apPlications for the Environment (PEOPLE) series of the Earth Observation Science for Society ([EO4Society](https://eo4society.esa.int/)) programme.
