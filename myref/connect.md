# CoRE Stack Backend Server Setup & Connection Guide

This guide explains how to connect to the remote **Turing Server** (`10.147.20.85` / `192.168.1.48`) hosted in the RISE Lab at IIT Madras, sync your local project, set up the environment, and run the backend services.

---

## 🗺️ 1. Connecting to the Server

You have two methods for connecting to the **Turing** server: **ZeroTier (Recommended)** or the **RISE Lab Gateway Proxy**.

### Method A: ZeroTier (Recommended)
ZeroTier creates a secure virtual private network (VPN) that allows direct peer-to-peer connection to the server.

1. **Join the network:**
   ```bash
   sudo zerotier-cli join 0cccb752f77865a1
   ```
2. **Get status and request approval:**
   Run the following command and share the output with `@kayceesrk` to get your device approved:
   ```bash
   sudo zerotier-cli status
   ```
3. **Verify the connection:**
   Once approved, verify that you are connected:
   ```bash
   sudo zerotier-cli listnetworks
   ```
   *You should see `0cccb752f77865a1 PrismLab @ IIT Madras` with status `OK` and a `10.147.20.x` IP address assigned to you.*
4. **SSH into Turing:**
   ```bash
   ssh snaveen@10.147.20.85
   ```

---

### Method B: Institute Gateway Proxy (Alternative)
If you are not using ZeroTier or are waiting for approval, you can tunnel through the RISE lab gateway (`14.139.160.85:443`).

> [!NOTE]
> This method requires your public SSH key (`~/.ssh/id_rsa.pub` or `~/.ssh/id_ed25519.pub`) to be added to `/home/random/.ssh/authorized_keys` on the gateway machine.

Connect using the proxy jump (`-J`) flag:
```bash
ssh -J random@14.139.160.85:443 snaveen@192.168.1.48
```

---

### ⚙️ Simplifying SSH with `~/.ssh/config`
To avoid typing long SSH commands every time, append the following to your **local** `~/.ssh/config` file:

```text
# Connection using ZeroTier
Host turing-zt
    HostName 10.147.20.85
    User snaveen

# Connection using the RISE Lab Gateway Proxy
Host turing-proxy
    HostName 192.168.1.48
    User snaveen
    ProxyJump random@14.139.160.85:443
```

Now you can connect simply by running:
```bash
ssh turing-zt
# OR
ssh turing-proxy
```

---

## 🔄 2. Syncing Your Project to the Server

Since your project runs on the server rather than your local machine, you need to copy the files to the remote server.

### Option 1: Using `rsync` (Recommended)
`rsync` is fast because it only copies files that have changed. Run this from your **local machine** terminal in the root directory containing the project:

* **Via ZeroTier:**
  ```bash
  rsync -avz --exclude '.git' --exclude 'venv' --exclude '.conda' --exclude '__pycache__' --exclude 'db.sqlite3' /Users/snaveen/Desktop/Core-stack-backend snaveen@10.147.20.85:~/
  ```
* **Via Gateway Proxy:**
  ```bash
  rsync -avz -e "ssh -o 'ProxyCommand ssh random@14.139.160.85 -p 443 -W %h:%p'" --exclude '.git' --exclude 'venv' --exclude '.conda' --exclude '__pycache__' --exclude 'db.sqlite3' /Users/snaveen/Desktop/Core-stack-backend snaveen@192.168.1.48:~/
  ```

### Option 2: Using `scp` (Alternative)
* **Via Gateway Proxy:**
  ```bash
  scp -o "ProxyCommand ssh random@14.139.160.85 -p 443 -W %h:%p" -r /Users/snaveen/Desktop/Core-stack-backend snaveen@192.168.1.48:~/
  ```

---

## 🛠️ 3. Setting Up the Environment on the Server

Once logged into the server (`turing`), navigate to your copied project directory and initialize the environment. Since the server runs Linux, you can utilize the automated script:

1. **Navigate to the directory:**
   ```bash
   cd ~/Core-stack-backend
   ```
2. **Run the automated installer:**
   The installation script will handle Python dependencies, PostgreSQL setup, RabbitMQ setup, migrations, and seed data.
   ```bash
   cd installation
   chmod +x install.sh
   ./install.sh
   ```
3. **Verify the Environment:**
   If you need to make changes to configuration or verify variables, check `nrm_app/.env`:
   ```bash
   nano nrm_app/.env
   ```

---

## 🚀 4. Running the Services on the Server

To start the server, open two separate shell sessions on the server (using terminal tabs or `tmux`).

### Terminal 1: Django Web Server
1. Activate the environment:
   ```bash
   conda activate corestack-backend
   ```
2. Start the Django development server binding to `0.0.0.0` (all interfaces) to allow remote access:
   ```bash
   python manage.py runserver 0.0.0.0:8000
   ```

### Terminal 2: Celery Worker
1. Activate the environment:
   ```bash
   conda activate corestack-backend
   ```
2. Start the Celery worker for handling computing tasks:
   ```bash
   celery -A nrm_app worker -l info -Q nrm
   ```

---

## 🌐 5. Accessing the Services From Your Local Browser

### Option A: Access directly via ZeroTier
If you are connected via ZeroTier, you can access the Django admin panel and APIs directly in your local browser:
* **Admin Interface:** `http://10.147.20.85:8000/admin/`
* **Swagger Documentation:** `http://10.147.20.85:8000/swagger/`

### Option B: Access via SSH Port Forwarding (Recommended)
If ZeroTier is unavailable or you want a secure direct tunnel, establish an SSH tunnel when connecting to forward remote port `8000` to local port `8000`.

* **Via ZeroTier Tunnel:**
  ```bash
  ssh -L 8000:localhost:8000 snaveen@10.147.20.85
  ```
* **Via Gateway Proxy Tunnel:**
  ```bash
  ssh -L 8000:localhost:8000 -J random@14.139.160.85:443 snaveen@192.168.1.48
  ```
* **Using `~/.ssh/config`:**
  You can also add `LocalForward 8000 localhost:8000` under your config profiles so it happens automatically!

Now, open your browser and navigate to:
* **Localhost URL:** `http://127.0.0.1:8000/`
