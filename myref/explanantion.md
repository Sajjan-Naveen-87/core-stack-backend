# CoRE Stack Backend

The **CoRE (Common Resource Engine) Stack Backend** is a Django-based geospatial and planning platform designed to support natural resource management (NRM), watershed planning, and rural development monitoring. It acts as the data-processing and orchestration engine that links field observations, remote sensing data, and geospatial visualization.

---

## What the Project Does

The system integrates geographical datasets, remote sensing processing pipelines, and field-level planning. Its core functionalities are:

1. **Geographical Administration (`geoadmin`)**:
   * Manages hierarchy lookup tables mapping official Government boundary codes (LG Directory/Census) and Survey of India (SOI) boundaries (State, District, Block/Tehsil, Gram Panchayat).

2. **Geospatial Analytics & Remote Sensing (`computing`)**:
   * Uses **Google Earth Engine (GEE)** and local computation pipelines to process rasters and calculate critical environmental parameters:
     * **Hydrology**: Run-off, precipitation, evapotranspiration, groundwater levels, and microwatershed (MWS) water budgets.
     * **Land Use / Land Cover (LULC)**: Multi-year land use classifications, terrain slope-classification, change detection, and cropping intensity.
     * **Tree Health**: Canopy cover density (CCD), canopy height, and vegetation anomalies.
   * Auto-publishes these computed layers into a **Geoserver** instance for mapping interfaces to render.

3. **Participatory Watershed Planning (`plans` & `dpr`)**:
   * Collects field survey data via **Open Data Kit (ODK)** and structures it into local project resources.
   * Builds planning modules allowing users to specify proposed NRM assets (recharge structures, irrigation systems, plantations) and sync them with ODK/S3.
   * Auto-generates **Detailed Project Reports (DPR)**, multi-watershed summary reports, and government formats (such as Yuktdhara formats) in PDF and Excel.

4. **Public Services & Integrations (`public_api` & `bot_interface`)**:
   * Exposes open API endpoints for lat/lon lookups to determine boundary codes or microwatershed identifiers.
   * Integrates webhook services for automated chatbot platforms (e.g., WhatsApp bot) to sync field feedback and query statuses.

---

## How It Is Used

The backend is used by developers, GIS analysts, field-workers, and dashboard interfaces:

### 1. By the Frontend / GIS Applications
* Renders geoserver-hosted layers for interactive map visualizations.
* Queries API endpoints under `/api/v1/public_api/` for lat/lon metadata mapping.

### 2. By Field Workers (Offline Sync)
* Field planners collect data using ODK templates.
* The system retrieves or receives XML submissions via `/api/v1/sync_offline_data/` to log and parse asset details.

### 3. By Celery Asynchronous Workers
* Processes resource-intensive GEE operations in the background via Celery tasks queued under `nrm`.

### 4. For Reporting
* Generates on-demand DPR reports via APIs under `/api/v1/generate_dpr/` to facilitate project compliance and approvals.
