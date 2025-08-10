### Production-ready monitoring, logging, and alerting for a single EC2

This stack deploys industry-standard tooling on one EC2 host using Docker:
- Prometheus: metrics collection and alerting rules (30d / 20GB retention by default)
- Alertmanager: alert routing (Slack/Email)
- Grafana: dashboards and alerting UI
- Loki: log storage
- Promtail: log collection from files under `/var/log` and `/var/log/app`
- Node Exporter: host metrics
- cAdvisor: container metrics (if you run containers)
- Blackbox Exporter: HTTP checks for your app endpoints

All services bind on the host network with sensible defaults.

---

### 1) Prerequisites on the EC2 host (Ubuntu 22.04+)

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Install Docker Engine + Compose plugin
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Optional: run docker as your user
sudo usermod -aG docker $USER
newgrp docker
```

Clone or copy this repository to the server, then:

```bash
cd Monitoring
cp env.example .env
# Edit .env and set a strong Grafana admin password
```

---

### 2) Configure integrations

- Prometheus blackbox target(s): edit `prometheus/prometheus.yml` and add your app URLs under the `blackbox` job `static_configs`.
- Application metrics (optional): if your app exposes `/metrics`, add its target to the `application-metrics` job in `prometheus/prometheus.yml`.
- Alerting routes:
  - Default email route is configured with placeholders. Update SMTP settings in `alertmanager/alertmanager.yml`.
  - Slack (optional): use `alertmanager/alertmanager.slack.example.yml` as a reference to enable Slack. Create `alertmanager/secrets/slack_webhook_url` with your webhook. Then update `alertmanager/alertmanager.yml` accordingly.
- Logs: ensure your app writes to `/var/log/app/*.log` (or adjust `promtail/promtail-config.yml`).

---

### 3) Bring up the stack

```bash
docker compose up -d

# Verify containers
docker compose ps
```

Default ports (host network):
- Grafana: http://SERVER_IP:3000
- Prometheus: http://SERVER_IP:9090
- Alertmanager: http://SERVER_IP:9093
- Loki: http://SERVER_IP:3100
- Node Exporter: http://SERVER_IP:9100/metrics
- cAdvisor: http://SERVER_IP:8082
- Blackbox Exporter: http://SERVER_IP:9115

If you use UFW/security groups, allow needed ports (or, better, reverse proxy with TLS and restrict SGs).

```bash
# Example (Grafana only)
sudo ufw allow 3000/tcp
```

---

### 4) Grafana setup

- Login at `http://SERVER_IP:3000` with the credentials set in `.env`.
- Data sources are auto-provisioned: Prometheus and Loki.
- Recommended dashboards (import by ID in Grafana > Dashboards > Import):
  - Node Exporter Full: 1860
  - Docker / Container: 193
  - Blackbox Exporter: 7587
  - Loki Logs: 13639

---

### 5) Alerting

Alerting rules are defined in `prometheus/rules/general.rules.yml`. Out of the box:
- TargetDown
- HighCPUUsage (>85% for 10m)
- HighMemoryUsage (>90% for 10m)
- HighDiskUsage (>90% for 10m)
- HighLoadAverage
- BlackboxProbeFailed (enable blackbox targets first)

Routing is handled by Alertmanager (`alertmanager/alertmanager.yml`). Configure Email, and optionally Slack using the example file.

Reload Prometheus after rule changes:

```bash
curl -X POST http://127.0.0.1:9090/-/reload
```

---

### 6) Logs with Loki + Promtail

- Promtail scrapes `/var/log/*.log` and `/var/log/app/*.log` by default.
- Query logs in Grafana: Explore > select Loki > use labels like `{job="app"}`.

To change paths or labels, edit `promtail/promtail-config.yml` and restart promtail:

```bash
docker compose restart promtail
```

---

### 7) Production hardening

- Put Grafana/Prometheus/Alertmanager behind a reverse proxy with TLS (Nginx/Traefik/Caddy) and IP/SSO restrictions.
- Restrict security groups to trusted IPs.
- Persistent volumes are configured; back up `grafana_data`, `prometheus_data`, and `loki_data`.
- Prometheus retention defaults to 30d or 20GB (whichever first). Adjust in `docker-compose.yml`.
- Consider running this stack on a separate monitoring EC2.

---

### 8) Lifecycle

```bash
# Stop
docker compose down

# Upgrade images
docker compose pull
docker compose up -d

# View logs (replace SERVICE with: prometheus | grafana | loki | promtail | alertmanager | cadvisor | blackbox-exporter | node-exporter)
docker compose logs -f SERVICE
```

---

### 9) Structure

```
.
├── docker-compose.yml
├── env.example
├── README.md
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── general.rules.yml
├── alertmanager/
│   ├── alertmanager.yml
│   ├── alertmanager.slack.example.yml
│   ├── templates/
│   │   └── notification.tmpl
│   └── secrets/            # create slack_webhook_url here (not in git)
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── blackbox/
│   └── blackbox.yml
└── grafana/
    └── provisioning/
        ├── datasources/datasource.yml
        └── dashboards/dashboards.yml
    └── dashboards/ (JSON files if you want to pre-provision)
```
