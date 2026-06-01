# Running CoRE Stack Backend Locally

This guide explains how to set up the dependencies, configure the environment, and run the CoRE Stack Backend server and background worker on your local machine.

---

## 📋 Prerequisites
Before running the application, make sure you have installed:
1. **Miniconda** (or Anaconda)
2. **PostgreSQL** (with PostGIS extension installed and running)
3. **RabbitMQ** (used as Celery's message broker)

---

## ⚙️ 1. Environment Setup

### 1.1 Conda Environment
Create and activate the virtual environment using the provided `environment.yml` configuration:
```bash
# Navigate to project root
cd /Users/snaveen/Desktop/Core-stack-backend

# Create environment from yml file
conda env create -f installation/environment.yml

# Activate environment
conda activate corestack-backend
```

### 1.2 Database Configuration
Create a PostgreSQL user and database by logging into your PostgreSQL shell first:
```bash
# Log in as the postgres default user
psql -U postgres
```
Then, execute the following SQL commands inside the `psql` prompt:
```sql
CREATE USER corestack_admin WITH PASSWORD 'corestack@123';
CREATE DATABASE corestack_db OWNER corestack_admin;
GRANT ALL PRIVILEGES ON DATABASE corestack_db TO corestack_admin;
\q
```
*Note: Make sure your PostgreSQL server has the PostGIS extension loaded.*

### 1.3 Local Configurations (`.env`)
Generate a `.env` file inside `nrm_app/` from the template:
```bash
cp .env.example nrm_app/.env
```
Open `nrm_app/.env` and update the database and credentials:
```env
DEBUG=True
SECRET_KEY=your_secret_key_here

DB_NAME=corestack_db
DB_USER=corestack_admin
DB_PASSWORD=corestack@123

# (Optional: Provide Google Earth Engine and S3 paths if utilizing spatial computation)
GEE_SERVICE_ACCOUNT_KEY_PATH=data/gee_confs/your-gee-key.json
```

---

## 🛠️ 2. Database Migrations & Initial Setup

With the environment activated and `.env` configured, apply the database schema:

```bash
# Apply Django migrations
python manage.py migrate

# Create a local admin user (Django superuser)
python manage.py createsuperuser
```

---

## 🚀 3. Running the Services

The application requires **two separate processes** to run simultaneously for local development.

### Terminal 1: Django API Server
Launch the primary web server:
```bash
conda activate corestack-backend
python manage.py runserver 127.0.0.1:8000
```
* The API endpoints will be accessible at: `http://127.0.0.1:8000/`
* The interactive API Swagger documentation: `http://127.0.0.1:8000/swagger/`
* The Admin Dashboard panel: `http://127.0.0.1:8000/admin/`

### Terminal 2: Celery Worker
Launch the background worker to handle asynchronous GIS calculations (LULC, Hydrology, CLART, etc.):
```bash
conda activate corestack-backend
celery -A nrm_app worker -l info -Q nrm
```

---

## 🧪 4. Verifying Installation
You can run the built-in initialization script to test database connectivity, credentials, and API response pathways:
```bash
python computing/misc/internal_api_initialisation_test.py
```
This script will verify your GEE setup, local storage directory paths, and run a mock block layer generation query.
