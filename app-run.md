# Running CoRE Stack Backend Locally

This guide explains how to set up the dependencies, configure the environment, and run the CoRE Stack Backend server and background worker on your local machine.

Choose the setup instructions matching your operating system:
* **Linux (Ubuntu / WSL2)**: Use the automated installer script (`install.sh`).
* **macOS (Native)**: Use Homebrew and manual configuration steps (recommended as the automated script is optimized for Linux package managers and requires Bash 4+).

---

## 💻 Option A: macOS Native Setup (Manual)

Because macOS default Bash is outdated (version 3.2, lacking associative array support) and the installer script uses Ubuntu's `apt-get` tool, follow these steps to set up the backend natively on macOS.

### A.1 Install System Dependencies via Homebrew
Open a new terminal window (**Terminal 1**) and run:
```bash
# Update Homebrew
brew update

# Install PostgreSQL (version 16 recommended) and RabbitMQ
brew install postgresql@16 rabbitmq git wget postgis
```

### A.2 Start PostgreSQL and RabbitMQ Services
In the same terminal window (**Terminal 1**), start the background services:
```bash
# Start PostgreSQL (version 16)
brew services start postgresql@16

# Start RabbitMQ
brew services start rabbitmq
```

### A.3 Configure PostgreSQL Database
By default, Homebrew PostgreSQL allows you to log in as your system user or as `postgres`. In **Terminal 1**, run the following to log in to the PostgreSQL prompt:
```bash
psql postgres
```
Inside the `psql` console, run the SQL commands to create the database role and DB:
```sql
CREATE USER corestack_admin WITH PASSWORD 'corestack@123';
CREATE DATABASE corestack_db OWNER corestack_admin;
ALTER USER corestack_admin WITH SUPERUSER;
\q
```

### A.4 Create Conda Environment
In **Terminal 1**, create and activate the virtual environment using the provided `environment.yml` configuration:
```bash
# Create environment from yml file
conda env create -f installation/environment.yml

# Activate environment
conda activate corestack-backend
```

### A.5 Initialize Environment Variables (`.env`)
In **Terminal 1**, generate the local `.env` configuration inside `nrm_app/` from the template:
```bash
cp .env.example nrm_app/.env
```
Open `nrm_app/.env` and verify the database configuration:
```env
DB_NAME=corestack_db
DB_USER=corestack_admin
DB_PASSWORD=corestack@123
```
*(Make sure to configure other required credentials such as GEE or GCS paths as needed).*

### A.6 Run Django Migrations & Seed Data
With the environment activated and `.env` configured, execute in **Terminal 1**:
```bash
# Run Django database migrations (migration files have been generated to resolve circular dependencies)
python manage.py migrate

# Load seed data
python manage.py loaddata installation/seed/seed_data.json

# Create a local admin user (Django superuser)
python manage.py createsuperuser
```

---

## 🐧 Option B: Linux (Ubuntu / WSL2) Setup (Automated)

The automated backend installer (`installation/install.sh`) handles Python, PostgreSQL, RabbitMQ, `nrm_app/.env`, migrations, seed data, and a test superuser.

### B.1 Run the Automated Installer
```bash
# Navigate to the installation directory
cd installation

# Make the script executable
chmod +x install.sh

# Run the installation script
./install.sh
```

### B.2 Optional Integrations Setup via Installer
You can target specific installer steps using the `--only` flag:
* **GEE Service Account**:
  ```bash
  bash installation/install.sh --only gee_configuration --gee-json /full/path/to/service-account.json
  ```
* **GCS Bucket**:
  ```bash
  bash installation/install.sh --only gcs_bucket_configuration --input gcs_bucket_name=your-gcs-bucket
  ```
* **GeoServer**:
  ```bash
  bash installation/install.sh --only initialisation_check --input geoserver_url=https://host/geoserver --input geoserver_username=admin --input geoserver_password=your-password
  ```

---

## 🔧 Useful Installer Controls (Linux/WSL2)

* **Show exact step names**:
  ```bash
  bash installation/install.sh --list-steps
  ```
* **Rebuild or update missing variables in `nrm_app/.env`**:
  ```bash
  bash installation/install.sh --only env_file
  ```
* **Rerun backend validation tests**:
  ```bash
  bash installation/install.sh --only initialisation_check
  ```

---

## 🚀 3. Starting the Services

To run the application locally, activate the conda environment and start both the Django server and Celery background worker in separate terminal windows.

### Terminal 1: Django API Server
```bash
# Activate environment
conda activate corestack-backend

# Run development server
python manage.py runserver 127.0.0.1:8000
```
* **API Endpoints**: `http://127.0.0.1:8000/`
* **Swagger Docs**: `http://127.0.0.1:8000/swagger/`
* **Django Admin Panel**: `http://127.0.0.1:8000/admin/`

### Terminal 2: Celery Worker
Launch the background worker (required for computing APIs):
```bash
# Activate environment
conda activate corestack-backend

# Start Celery worker
celery -A nrm_app worker -l info -Q nrm
```

---

## 🔑 4. Log in and Invoke APIs

Computing APIs use JWT bearer tokens for authentication. Follow these steps to log in and make API calls:

### 1. Retrieve or Create a Superuser
Use the installer-generated superuser or create one manually with:
```bash
python manage.py createsuperuser
```

### 2. Obtain a JWT Access Token
Log in via a POST request:
```bash
curl -s -X POST http://127.0.0.1:8000/api/v1/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"your_username","password":"your_password"}'
```
This returns an `access` token. Save this token for subsequent requests.

### 3. Get the GEE Account ID
List the configured GEE accounts to find the correct numeric ID:
```bash
curl -s http://127.0.0.1:8000/api/v1/geeaccounts/ \
  -H "Authorization: Bearer <your-access-token>"
```

### 4. Trigger a Computing API Job
Keep both Django and Celery services running and make a request to trigger processing (e.g., LULC for a block):
```bash
curl -X POST http://127.0.0.1:8000/api/v1/lulc_for_tehsil/ \
  -H "Authorization: Bearer <your-access-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "state": "karnataka",
    "district": "raichur",
    "block": "devadurga",
    "start_year": 2022,
    "end_year": 2023,
    "gee_account_id": 1
  }'
```

---

## 🦊 5. Setup Headless Firefox for Selenium

To support scraping, headless PDF generation, or automated browser actions using Selenium:

### 5.1 Install Firefox ESR
On your local Ubuntu/Linux host:
```bash
sudo apt-get update
sudo apt-get install -y firefox-esr
```

### 5.2 Verify Headless Browser Run
Verify Firefox can start and capture a page in headless mode:
```bash
/usr/bin/firefox-esr --headless --screenshot /tmp/ff_test.png https://example.com/
```

### 5.3 Install Webdriver Manager & Dependencies
Install the package to automatically fetch and manage correct Geckodriver versions:
```bash
python -m pip install webdriver-manager==4.0.2
```

### 5.4 Run Smoke Tests
1. **Start Geckodriver pointing at Firefox ESR**:
   ```bash
   "$HOME/.wdm/drivers/geckodriver/linux64/v0.36.0/geckodriver" \
     --log trace \
     --binary /usr/bin/firefox-esr
   ```
2. **Execute a verification script in Python**:
   ```python
   from selenium.webdriver import Remote
   from selenium.webdriver.firefox.options import Options

   opts = Options()
   opts.headless = True
   
   driver = Remote(command_executor="http://127.0.0.1:4444", options=opts)
   driver.get("https://example.com")
   print("Title:", driver.title)  # Expected: Title: Example Domain
   driver.quit()
   ```
