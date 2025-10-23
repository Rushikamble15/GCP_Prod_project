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
# DevOps Deployment: Flask Application on Google Cloud
Infrastructure-as-Code–style guide for deploying a Flask web application on Google Cloud Platform (GCP) using private VMs, Cloud NAT, Managed Instance Groups (MIGs), and a Regional HTTP(S) Load Balancer. A domain from GoDaddy is integrated with Google Cloud DNS for custom domain routing.

---

## Table of contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Reference network & IP plan](#reference-network--ip-plan)
- [Step 1 — Create VPC and Subnets](#step-1--create-vpc-and-subnets)
- [Step 2 — Configure Cloud Router and Cloud NAT](#step-2--configure-cloud-router-and-cloud-nat)
- [Step 3 — Firewall rules](#step-3--firewall-rules)
- [Step 4 — Instance template (Flask app)](#step-4--instance-template-flask-app)
- [Step 5 — Managed Instance Group (MIG)](#step-5--managed-instance-group-mig)
- [Step 6 — Regional External HTTP(S) Load Balancer](#step-6--regional-external-https-load-balancer)
- [Step 7 — Domain setup (GoDaddy) + Cloud DNS](#step-7--domain-setup-godaddy--cloud-dns)
- [Step 8 — Validation](#step-8--validation)
- [Troubleshooting & notes](#troubleshooting--notes)
- [Cleanup](#cleanup)

---

## Overview
This guide shows how to host a simple Flask app on private Compute Engine instances (no external IPs) in a VPC. Instances use Cloud NAT for outbound access. A regional HTTP(S) load balancer exposes the app to the internet while keeping backends private. A proxy-only subnet is included for the load balancer's managed proxy resources.

Recommended region example: `asia-south1` (pick one region and use it consistently).

---

## Prerequisites
- Google Cloud project with billing enabled.
- Owner or Editor permissions (or equivalent) to create networks, Compute Engine resources, Cloud NAT, and Cloud DNS.
- GoDaddy domain (or any domain registrar).
- Basic familiarity with the GCP Console.
- Optional: a bastion VM in a public subnet for debugging and curl checks.

---

## Reference network & IP plan
- VPC name: `prod-vpc`
- Subnets (examples):
  - `subnet-public` — 10.10.10.0/24 (for bastion, optional)
  - `subnet-app` — 10.10.20.0/24 (private instances)
  - `subnet-proxy-only` — 10.10.30.0/24 (proxy-only for regional LB)
- Healthcheck port: 8080
- Flask app listens on port 8080

---

## Step 1 — Create VPC and Subnets
1. Console: VPC network → VPC networks → Create VPC network
   - Name: `prod-vpc`
   - Subnet creation mode: `Custom`
   - Add subnets:
     - `subnet-public` — 10.10.10.0/24
     - `subnet-app` — 10.10.20.0/24
2. Create proxy-only subnet:
   - VPC → `prod-vpc` → Subnetworks → Create subnetwork
   - Name: `subnet-proxy-only`
   - Region: your region (e.g., `asia-south1`)
   - IP range: `10.10.30.0/24`
   - Purpose: `Regional Managed Proxy (proxy-only)`

---

## Step 2 — Configure Cloud Router and Cloud NAT
1. Network services → Cloud NAT → Create NAT gateway
   - Name: `nat-asia-south1`
   - Region: your chosen region
   - Network: `prod-vpc`
   - Cloud Router: Create new → `router-asia-south1`
   - NAT IPs: Auto-allocate (or specify reserved static IPs if required)
   - Subnetworks: Add `subnet-app` (and optionally `subnet-public` if your bastion needs NAT)
2. Create and apply. Cloud NAT allows private instances to reach the internet (for package updates, container pulls, etc.) without external IPs.

---

## Step 3 — Firewall rules
Create the following firewall rules (VPC → Firewall rules → Create firewall rule):

- Allow Google Load Balancer health checks (ingress)
  - Name: `allow-health-checks-8080`
  - Direction: Ingress
  - Source ranges: `35.191.0.0/16`, `130.211.0.0/22`
  - Protocols/ports: TCP:8080
  - Target: Instances tagged or all in `prod-vpc` as appropriate

- Allow proxy-only subnet -> app subnet traffic (ingress)
  - Name: `allow-proxy-only-8080`
  - Direction: Ingress
  - Source range: `10.10.30.0/24`
  - Protocols/ports: TCP:8080
  - Target: Instances in `subnet-app` or with matching tags

Notes:
- Restrict target instances by network tags where appropriate (e.g., `flask-backend`) so rules don't apply to unrelated VMs.
- If you use HTTPS, allow TCP:443 as needed between proxy and backend.

---

## Step 4 — Instance template (Flask app)
Compute Engine → Instance templates → Create instance template
- Name: `tmpl-flask-app`
- Machine type: `e2-micro` or `e2-small` (adjust for production)
- Boot disk: Debian/Ubuntu LTS
- Network: `prod-vpc`
- Subnet: `subnet-app`
- External IP: None
- Add network tag (optional): `flask-backend` (use in firewall rules)
- Metadata / Startup script: paste the startup script below

Startup script (runs at boot, creates a venv, installs Flask+Gunicorn, creates a systemd service):
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

# Systemd service for Flask via Gunicorn
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
```

Notes:
- The service uses Gunicorn with 2 workers (adjust `-w` as needed).
- Ensure the VM image supports systemd (Debian/Ubuntu do).

---

## Step 5 — Managed Instance Group (MIG)
Compute Engine → Instance groups → Create instance group
- Name: `mig-flask-app`
- Location: Single zone (e.g., `asia-south1-a`) or zonal as needed
- Instance template: `tmpl-flask-app`
- Autoscaling: set min = 2, max = 4 (demo values; tune for traffic)
- Health checks: the load balancer's health check will be used later (or create a basic group-level health check on port 8080)
- Create.

Optional verification (via bastion in `subnet-public`):
- SSH to bastion (has external IP)
- From bastion: `ssh <internal-ip-of-instance>` then `curl http://127.0.0.1:8080/health`

---

## Step 6 — Regional External HTTP(S) Load Balancer
Network services → Load balancing → Create load balancer
- Choose: Application Load Balancer (HTTP(S))
- From Internet to my VMs or serverless services → Start configuration
- Scope: Regional (choose same region)

Frontend:
- Protocol: HTTP (or HTTPS if you configure a certificate)
- Port: 80 (or 443 for HTTPS)
- IP address: Create/select External IPv4 (e.g., `flask-ip`)
- VPC network for forwarding rule: `prod-vpc`

Backend:
- Backend type: Instance group
- Instance group: `mig-flask-app`
- Backend port: 8080
- Health check: Create new → HTTP port 8080 → Request path: `/health`

Routing:
- Default URL map → point to the backend

Proxy-only subnet:
- When configuring a regional load balancer, specify the `subnet-proxy-only` in the same region/VPC so the managed proxy resources have the proxy-only subnet available.

After provisioning, the frontend IP should return the Flask response:
- http://<LB_IP>/ → "Hello from Flask on GCP (private subnet)!"
- http://<LB_IP>/health → ok

If you need HTTPS, configure an SSL certificate via Google-managed certificates or upload your certificate, and add an HTTPS frontend.

---

## Step 7 — Domain setup (GoDaddy) + Cloud DNS
High-level flow:
1. Create a Cloud DNS public zone:
   - Network services → Cloud DNS → Create zone
   - Zone type: Public
   - Zone name: e.g., `example-in-zone`
   - DNS name: `example.in.`

2. Copy the 4 Cloud DNS nameservers (ns-cloud-*.googledomains.com).

3. In GoDaddy (domain registrar):
   - Go to your domain's Nameservers settings → choose "Custom nameservers"
   - Enter the 4 Cloud DNS nameservers and save (delegates domain to Cloud DNS)

4. Add records in Cloud DNS after LB gets its external IP:
   - Create an A record (apex):
     - Type: A
     - Name: @ (or leave blank)
     - TTL: 300
     - IPv4 address: <LB_IP>
   - Optional: CNAME for www:
     - Type: CNAME
     - Name: www
     - Canonical name: example.in.

Notes:
- DNS propagation can take some time. Use dig to check:
  - dig A example.in +short
  - dig NS example.in +short

---

## Step 8 — Validation
By IP (before DNS resolves):
- Open in browser: http://<LB_IP>/
- Health: http://<LB_IP>/health

By domain (after DNS delegation + A record):
- Open in browser: http://example.in/
- Health: http://example.in/health

Terminal checks:
- dig A example.in +short
- dig NS example.in +short
- curl -i http://example.in/

---

## Troubleshooting & notes
- Health checks failing:
  - Ensure firewall allows health check source ranges: `35.191.0.0/16` and `130.211.0.0/22`.
  - Confirm the backend listens on the configured port (8080) and the health path `/health` returns 200.
- Connectivity from private instances:
  - Ensure Cloud NAT is configured with the right subnetwork (subnet-app).
  - Check instance logs for package installation errors in the startup script: `journalctl -u flask.service` or `systemctl status flask.service`.
- Firewall scoping:
  - Use network tags (e.g., `flask-backend`) to scope firewall rules to only backend instances.
- HTTPS:
  - Use Google-managed certificates (Network services → Load balancing → Certificate manager) for automatic renewal.
- Logging and monitoring:
  - Enable Cloud Logging and Cloud Monitoring for the MIG and Load Balancer to see detailed health and request metrics.

---

## Cleanup
To avoid charges, delete resources when done:
- Delete load balancer (frontend, URL map, backend service, forwarding rule)
- Delete managed instance group and instance templates
- Delete instance templates, disks, and VM instances
- Delete Cloud NAT and Cloud Router
- Delete Cloud DNS zone and revert nameservers in your registrar
- Delete VPC network

