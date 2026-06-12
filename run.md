# CoRE Stack Backend: Local Run Guide

Follow these steps to spin up and run the CoRE Stack Backend locally. Make sure your virtual environment (`corestackenv`) is active where necessary.

---

### Terminal 1: Infrastructure & Services (Docker / System)

Ensure PostgreSQL, RabbitMQ, and GeoServer are active.

```bash
# 1. Start the Docker container for GeoServer
sudo docker start geoserver

# 2. Verify PostgreSQL is running
sudo systemctl status postgresql

# 3. Verify RabbitMQ is running
sudo systemctl status rabbitmq-server
```

*GeoServer will be accessible at: `http://localhost:8080/geoserver` (credentials: `admin` / `geoserver`).*

---

### Terminal 2: Celery Worker

The Celery worker processes asynchronous GIS tasks and GEE pipeline jobs.
**Important**: The queue must listen to `nrm` (where GIS tasks are routed) and the default `celery` queue.

```bash
# 1. Activate the environment
conda activate corestackenv

# 2. Run the Celery worker listening to both 'nrm' and 'celery' queues
celery -A nrm_app worker -l info -Q nrm,celery
```

---

### Terminal 3: Django Web API Server

Starts the primary REST API endpoint service.

```bash
# 1. Activate the environment
conda activate corestackenv

# 2. Start the Django dev server
python manage.py runserver 0.0.0.0:8000 --noreload
```

*The API endpoints will be accessible at: `http://localhost:8000`.*

---

### Terminal 4: Verification & Diagnostic Checks

To verify that everything is running correctly, run the internal API initialization checks:

```bash
# 1. Activate the environment
conda activate corestackenv

# 2. Run the validation checks
python computing/misc/internal_api_initialisation_test.py --require-gee
```

*Expected output: `Internal API initialisation test passed.`*

---

## 🚀 LULC Pipeline Execution Guide

Run these steps sequentially from a terminal client to generate block boundaries, microwatershed layers, and the LULC layer.

### Step 1: Obtain a JWT Authentication Token

Submit your superuser credentials to get the access token. Replace the password if modified.

```bash
curl -s -X POST http://localhost:8000/api/v1/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"test_user_4982","password":"admin"}'
```

Copy the `"access"` token value from the JSON response to use in the following commands as `$TOKEN`.

### Step 2: Generate the Block Boundary Layer

Trigger the admin boundary creation. This processes the local geojson files and registers them.

```bash
curl -s -X POST http://localhost:8000/api/v1/generate_block_layer/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "state": "assam",
    "district": "baksa",
    "block": "baksa",
    "gee_account_id": 1
  }'
```

*Verify in the Celery worker terminal that the task completes successfully.*

### Step 3: Generate the Microwatershed (MWS) Layer

Generates microwatershed boundaries. This is a mandatory dependency for the LULC layer, as it defines the spatial boundaries GEE uses to clip classification rasters.

```bash
curl -s -X POST http://localhost:8000/api/v1/generate_mws_layer/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "state": "assam",
    "district": "baksa",
    "block": "baksa",
    "gee_account_id": 1
  }'
```

*Wait until the Celery log prints: `Task computing.mws.mws.mws_layer[...] succeeded`.*

### Step 4: Generate the LULC Layer

Trigger the Land Use / Land Cover classification. This performs satellite-based LULC pixel extraction inside Google Earth Engine.

```bash
curl -s -X POST http://localhost:8000/api/v1/lulc_v3/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "state": "assam",
    "district": "baksa",
    "block": "baksa",
    "start_year": 2023,
    "end_year": 2024,
    "gee_account_id": 1
  }'
```

*This will submit cloud classification tasks to Earth Engine. Progress will be printed in the Celery worker logs every 60 seconds as the task polls GEE.*
