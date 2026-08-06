---
title: Docker Services — IP Config & App Deployment (Modified Topology)
tags:
  - lab/docker
  - reference/services
---

# Docker Services — IP Config & App Deployment (Modified Topology)

## Network Layout

| Service | Connected To | Subnet | Assigned IP |
|---|---|---|---|
| Docker Router eth0 | OVS-1 (LAN1) | 192.168.10.0/24 | 192.168.10.254 |
| Docker Router eth1 | OVS-2 (LAN2) | 192.168.20.0/24 | 192.168.20.254 |
| DHCP-Server (Alpine dnsmasq) | OVS-2 (LAN2) | 192.168.20.0/24 | 192.168.20.2 |
| Ubuntu Desktop | OVS-2 (LAN2) | 192.168.20.0/24 | DHCP |
| Traffic-Gen-1 | OVS-2 (LAN2) | 192.168.20.0/24 | 192.168.20.10 |
| Grafana-1 | OVS-2 (LAN2) | 192.168.20.0/24 | 192.168.20.11 |
| Prometheus-1 | OVS-2 (LAN2) | 192.168.20.0/24 | 192.168.20.12 |
| webterm-1 | OVS-1 (LAN1) | 192.168.10.0/24 | DHCP |
| appServer-1 | OVS-1 (LAN1) | 192.168.10.0/24 | 192.168.10.10 |
| PostgreSQL-1 | OVS-1 (LAN1) | 192.168.10.0/24 | 192.168.10.11 |
| PC1 | OVS-1 (LAN1) | 192.168.10.0/24 | DHCP |

## Step 0 — Build Docker Router

```bash
docker build -t docker-router:latest - << 'EOF'
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y iproute2 iptables && rm -rf /var/lib/apt/lists/*
CMD ["sh", "-c", "\
    ip addr add 192.168.10.254/24 dev eth0 && \
    ip addr add 192.168.20.254/24 dev eth1 && \
    ip link set eth0 up && \
    ip link set eth1 up && \
    echo 1 > /proc/sys/net/ipv4/ip_forward && \
    echo 'nameserver 8.8.8.8' > /etc/resolv.conf && \
    tail -f /dev/null"]
EOF
```

GNS3 template: 2 adapters, Telnet console.

## Step 1 — Start Docker Nodes

In GNS3 GUI, start each Docker node one by one. If they stay "Created" status:
1. Right-click → Stop
2. Right-click → Delete
3. Right-click in topology → Add → Docker container → Re-select the template

## Step 2 — Configure PostgreSQL

Console into PostgreSQL-1:

```bash
# Set IP
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1

# Start PostgreSQL
su - postgres -c "pg_ctl -D /var/lib/postgresql/data start"

# Create database and user
su - postgres -c "psql -c \"CREATE DATABASE labdb;\""
su - postgres -c "psql -c \"CREATE USER labuser WITH PASSWORD 'gns3lab';\""
su - postgres -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE labdb TO labuser;\""

# Verify
su - postgres -c "psql -c '\\l'"
```

## Step 3 — Configure App Server

Console into appServer-1:

```bash
# Set IP
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
echo nameserver 8.8.8.8 > /etc/resolv.conf

# Install Flask
pip install flask psycopg2-binary requests --break-system-packages

# Create the app
cat > /opt/app.py << 'EOF'
from flask import Flask, request
import psycopg2, os

app = Flask(__name__)

DB_HOST = os.getenv("DB_HOST", "192.168.10.11")
DB_NAME = os.getenv("DB_NAME", "labdb")
DB_USER = os.getenv("DB_USER", "labuser")
DB_PASS = os.getenv("DB_PASS", "gns3lab")

@app.route("/")
def home():
    return "<h1>App Server</h1><p>Connected to PostgreSQL</p>"

@app.route("/health")
def health():
    return {"status": "ok", "service": "app-server"}

@app.route("/db-check")
def db_check():
    try:
        conn = psycopg2.connect(host=DB_HOST, dbname=DB_NAME, user=DB_USER, password=DB_PASS)
        cur = conn.cursor()
        cur.execute("SELECT 1")
        return {"database": "connected", "result": cur.fetchone()[0]}
    except Exception as e:
        return {"database": "error", "message": str(e)}, 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
EOF

# Run
python3 /opt/app.py &
```

Verify: `curl http://192.168.10.10/` should return the HTML.
`curl http://192.168.10.10/db-check` should return `{"database": "connected"}`.

## Step 4 — Configure Grafana

Console into Grafana-1:

```bash
# Set IP
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

Access: `http://192.168.20.11:3000` (from LAN2).
Default login: `admin` / `admin`.

### Add Prometheus datasource
1. Login to Grafana at `http://192.168.20.11:3000`
2. Connections → Add data source → Prometheus
3. URL: `http://192.168.20.12:9090`
4. Save & Test

## Step 5 — Configure Prometheus

Console into Prometheus-1:

```bash
# Set IP
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

Access: `http://192.168.20.12:9090` (from LAN2).

## Step 6 — Configure Traffic-Gen-1

Console into Traffic-Gen-1:

```bash
# Set IP
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
echo nameserver 8.8.8.8 > /etc/resolv.conf

# Install tools
apk add --no-cache curl busybox-extras

# Generate traffic to various targets
while true; do
    curl -s http://192.168.10.10/ > /dev/null 2>&1
    curl -s http://192.168.10.10/health > /dev/null 2>&1
    ping -c 1 192.168.10.1 > /dev/null 2>&1
    sleep 5
done &
```

## Step 7 — Verify Cross-LAN Routing (Docker Router)

```bash
# From PC1 (LAN1):
PC1> ping 192.168.20.1
PC1> ping 192.168.20.11

# From Ubuntu (LAN2):
ping 192.168.10.1
ping 192.168.10.10
```

## Step 8 — Verify End-to-End

| Test | Command | Expected |
|---|---|---|
| App server reachable | `curl http://192.168.10.10/` | HTML response |
| DB connectivity | `curl http://192.168.10.10/db-check` | `{"database": "connected"}` |
| Grafana UI | Browser to `http://192.168.20.11:3000` | Login page |
| Prometheus UI | Browser to `http://192.168.20.12:9090` | UI |
| Cross-LAN ping | From PC1: `ping 192.168.20.10` | Reachable via Docker Router |
| Internet on LAN1 | From PC1: `ping 8.8.8.8` | Reachable via FGT-Primary |
| Internet on LAN2 | From Ubuntu: `ping 8.8.8.8` | Reachable via FGT-Secondary |
