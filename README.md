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



Step 5 — Create the Managed Instance Group (MIG)
Compute Engine → Instance groups → Create instance group
Name: mig-flask-app
Location: Single zone (e.g., asia-south1-a)
Instance template: tmpl-flask-app
Autoscaling: Min 2, Max 4 (demo)
Create.
Verification (optional, via a bastion in subnet-public): you can SSH to a bastion with an external IP, then SSH to private instances using internal IP, and curl http://:8080/health

Step 6 — Create the Regional External HTTP(S) Load Balancer
Network services → Load balancing → Create load balancer

Choose: Application Load Balancer
From Internet to my VMs or serverless services → Start configuration
Load balancer scope: Regional
Region: your chosen region
Continue.
Frontend (HTTP for now)

Protocol: HTTP
Port: 80
IP address: Create or select a new External IPv4 (e.g., flask-ip)
VPC network for forwarding rule: prod-vpc
Save.
Backend

Backend type: Instance group
Instance group: mig-flask-app
Port: 8080
Health check: Create new → HTTP on port 8080 → Request path /health
Save.
Routing

Default URL map → points to the backend above
Proxy-only subnet

Ensure your Regional LB references your proxy-only subnet in the same region/VPC (we created it in Step 1).
Review & Create

After provisioning, open the frontend IP in a browser:
You should see: Hello from Flask on GCP (private subnet)!
Step 7 — Domain setup (GoDaddy high-level) + Cloud DNS records
High-level in GoDaddy:

Buy or use an existing domain (e.g., example.in)
In the domain’s DNS/Nameservers panel, choose “Custom nameservers”
You will set these to the 4 nameservers that Google gives you in Cloud DNS (next steps)
Create a Cloud DNS public zone:

Network services → Cloud DNS → Create zone
Zone type: Public
Zone name: example-in-zone (any friendly name)
DNS name: example.in.
Create.
Copy the 4 nameservers (ns-cloud-*.googledomains.com).
Add records in Cloud DNS (after the LB is ready and has an external IP):

Add an A record at the apex:

Add standard → Type: A
Name: (leave blank or use @)
TTL: 300
IPv4 address:
Save.
Optional CNAME for www:

Type: CNAME
Name: www
Canonical name: example.in.
TTL: 300
Save.
Delegate the domain to Google in GoDaddy:

In GoDaddy → Nameservers → set to the 4 Cloud DNS nameservers you copied
Save. Propagation can take some time (usually minutes, sometimes longer)
Step 8 — Validate
By IP (before DNS):

Open http://<LB_IP>/
Open http://<LB_IP>/health → should return ok
By domain (after delegation + A record):

Open http://example.in/
Open http://example.in/health → should return ok
Optional terminal checks (any machine with dig/curl):

dig A example.in +short
dig NS example.in +short
curl -i http://example.in/

