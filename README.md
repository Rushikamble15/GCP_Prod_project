# DevOps Deployment: Flask Application on Google Cloud with Private Subnets, Cloud NAT, and Regional HTTP(S) Load Balancer

> Infrastructure-as-Code–style deployment of a Flask web application on Google Cloud Platform (GCP) using private VMs, Cloud NAT, Managed Instance Groups (MIGs), and a Regional External HTTP(S) Load Balancer — designed for DevOps engineers.

---

## 📋 Overview

This project demonstrates how to deploy a **Flask application** on **Google Cloud Platform** using a **secure and scalable architecture**.  
The setup includes **private subnets**, **Cloud NAT** for outbound access, **Managed Instance Groups** for auto-scaling, and a **Regional HTTP(S) Load Balancer** for traffic distribution.  
A **GoDaddy domain** is integrated with **Google Cloud DNS** for custom domain routing.

---

## ⚙️ Prerequisites

- Google Cloud project with billing enabled  
- Basic familiarity with the GCP Console (UI)  
- GoDaddy domain (or new domain purchase)  
- Consistent region (e.g., `asia-south1`)

---

## 🚀 Step 1 — Create VPC and Subnets

1. **VPC network → VPC networks → Create VPC network**
   - Name: `prod-vpc`
   - Subnet creation mode: `Custom`
   - Add subnets:
     - `subnet-public`: `10.10.10.0/24`
     - `subnet-app`: `10.10.20.0/24`
   - Click **Create**

2. **Create Proxy-only Subnet**
   - VPC → `prod-vpc` → Subnetworks → **Create subnetwork**
   - Name: `subnet-proxy-only`
   - Region: your region (e.g., `asia-south1`)
   - IP range: `10.10.30.0/24`
   - Purpose: `Regional Managed Proxy (proxy-only)`
   - Click **Create**

---

## 🌐 Step 2 — Configure Cloud Router and Cloud NAT

1. **Network services → Cloud NAT → Create NAT gateway**
   - Name: `nat-asia-south1`
   - Region: your region
   - Network: `prod-vpc`
   - Cloud Router: Create new → `router-asia-south1`
   - NAT IPs: Auto-allocate
   - Subnetworks: Select `subnet-app` (and optionally `subnet-public`)
   - Click **Create**

---

## 🔥 Step 3 — Firewall Rules

### Allow Google Health Checks
- Name: `allow-health-checks-8080`
- Direction: Ingress  
- Source ranges: `35.191.0.0/16`, `130.211.0.0/22`  
- Protocol/port: `TCP:8080`

### Allow Proxy-only Subnet Traffic
- Name: `allow-proxy-only-8080`
- Direction: Ingress  
- Source range: `10.10.30.0/24`  
- Protocol/port: `TCP:8080`

---

## 🐍 Step 4 — Create Instance Template (Flask App)

**Compute Engine → Instance templates → Create instance template**

- Name: `tmpl-flask-app`
- Machine type: `e2-micro` or `e2-small`
- Boot disk: Debian/Ubuntu LTS
- Network: `prod-vpc`
- Subnet: `subnet-app`
- External IP: None (private)
- **Startup script:**

```bash
#!/bin/bash
set -e

# Update & install prerequisites
apt-get update -y
apt-get install -y python3-pip python3-venv

# Create app user
id -u appuser &>/dev/null || useradd -m -s /bin/bash appuser

# Set up Flask app environment
sudo -u appuser bash <<'EOF'
cd ~
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install flask gunicorn

cat > app.py <<PY
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "Hello from Flask on GCP (private subnet)!"

@app.route("/health")
def health():
    return "ok", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
PY
EOF

# Systemd service for Flask
cat >/etc/systemd/system/flask.service <<'UNIT'
[Unit]
Description=Flask via Gunicorn
After=network.target

[Service]
User=appuser
WorkingDirectory=/home/appuser
ExecStart=/home/appuser/venv/bin/gunicorn -w 2 -b 0.0.0.0:8080 app:app
Restart=always

[Install]
WantedBy=multi-user.target
UNIT

systemctl daemon-reload
systemctl enable --now flask.service
