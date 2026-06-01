# Exploration & Feature Implementation Dashboard

This document serves as a unified reference dashboard for understanding the scientific resources behind the CoRE Stack project and tracking future feature implementations.

---

## 🔗 Part 1: Scientific & Project References

### A. CoRE Stack Platform
*   **[CoRE Stack Official Documentation](https://docs.core-stack.org/)**: Central guide for developers detailing setup, installation, and deployment.
*   **[CoRE Stack API Documentation](https://api-doc.core-stack.org/)**: Interactive Swagger portal to explore views, request parameters, and response schemas.
*   **[CoRE Stack Database Design](https://github.com/core-stack-org/core-stack-backend/wiki/DB-Design)**: Wiki containing PostgreSQL tables, constraints, and architecture mappings.
*   **[Integrating Custom Pipelines Guide](https://docs.google.com/document/d/1lfx2hJKndmzVp55ZHIIFYqRTz-8fZCWc9QikUDQpTN0/edit?usp=sharing)**: Developer Google Doc outlining how to build and register new Celery tasks with Google Earth Engine pipelines.

### B. Hydrology & Terrain Modeling
*   **[SCS-CN Runoff Method Reference](https://www.wcc.nrcs.usda.gov/ftpref/wntsc/H&H/NEHhydrology/ch10.pdf)**: USDA National Engineering Handbook chapter on the Soil Conservation Service Curve Number method used for runoff estimations.
*   **[JAXA GSMaP Precipitation (GEE Catalog)](https://developers.google.com/earth-engine/datasets/catalog/JAXA_GPM_L3_GSMaP_v6_operational)**: Earth Engine page for JAXA hourly satellite rainfall data.
*   **[NASA FLDAS Land Data Assimilation System](https://ldas.gsfc.nasa.gov/fldas)**: Information page on NASA's FEWS NET model providing soil moisture and evapotranspiration data.
*   **[Google Dynamic World LULC (GEE Catalog)](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_DYNAMICWORLD_V1)**: 10m deep-learning land cover dataset documentation.
*   **[FABDEM Global Elevation (GEE Catalog)](https://developers.google.com/earth-engine/datasets/catalog/projects_sat-io_open-datasets_FABDEM)**: 30-meter elevation model with forest and building heights removed, used for slope percentages.

### C. Participatory Planning (CLART)
*   **[India Observatory Portal](https://www.indiaobservatory.org.in/)**: Technological platform built by the Foundation for Ecological Security (FES) host to spatial decision tools.
*   **[Foundation for Ecological Security (FES India)](https://fes.org.in/)**: The non-profit leading ecological restoration and participatory NRM programs.
*   **[CLART App Concept Overview (YouTube Video)](https://www.youtube.com/watch?v=0k5G9_U-aF8)**: Video explanation of translating geo-hydrological layers (lithology, slope, lineaments) into village-level watershed recommendations.

---

## 🚀 Part 2: Feature Implementation Backlog

The following features are proposed to scale the CoRE Stack project into a collaborative open-data and sponsorship network. For full maps and architectures, refer to the [Future Feature Roadmap](file:///Users/snaveen/Desktop/Core-stack-backend/myref/extra.md).

### 1. Advanced Data Exchange
*   `[ ]` **Open-GIS STAC Catalog**: Expose calculated spatial layers via a standard SpatioTemporal Asset Catalog (STAC) API.
*   `[ ]` **IoT Well Telemetry Receiver**: Build webhook handlers to ingest real-time data from deployed cellular piezometer sensors.
*   `[ ]` **Secure Org-to-Org Data Rooms**: Allow tenant organizations to share shapefile layers privately for audits.

### 2. Next-Gen Mobile Collection
*   `[ ]` **Offline Mobile AI Classifier**: Train and integrate an offline TensorFlow Lite model on the ODK client app for automatic crop and soil degradation identification.
*   `[ ]` **Augmented Reality (AR) Overlay**: Add a camera overlay showing a 3D model of recommended check-dams on the physical terrain to verify slope compatibility.
*   `[ ]` **Citizen Science Gamification**: Launch a gamified points leaderboard for local farmers logging rain metrics.

### 3. Digital Sponsorship & Donations
*   `[ ]` **"Sponsor a Structure" Map**: Develop an interactive public portal exposing proposed CLART assets for direct CSR funding.
*   `[ ]` **eROI Dashboard (Ecological ROI)**: Connect funded coordinates with GEE time-series (VCI/NDVI) to automatically generate ecological impact reports.
*   `[ ]` **Compute & Dataset Donations**: Allow institutions to donate labeled GIS polygon datasets or share GEE cloud computing quotas.
