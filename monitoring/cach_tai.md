# Monitoring Stack Setup

This project runs:
 
- FastNetMon exporter metrics on port `9209`
- node_exporter on port `9100`
- Prometheus to scrape both targets
- Grafana to visualize metrics

## Quick Start

Create the files `docker-compose.yml` and `prometheus.yml`, then run:

```bash
docker-compose up -d
```

## 1. Install FastNetMon on Ubuntu Server

Update packages and install helper tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl
```

Download and install FastNetMon Community Edition:

```bash
wget https://install.fastnetmon.com/installer -O installer
sudo chmod +x installer
sudo ./installer -install_community_edition
```

Complete the FastNetMon setup in the installer if it asks for an email or license key.

Edit the FastNetMon config:

```bash
sudo nano /etc/fastnetmon.conf
```

Recommended Prometheus settings:

```ini
enable_ban = off
prometheus = on
prometheus_port = 9209
prometheus_host = 0.0.0.0
process_ban = off
```

Restart and enable the service:

```bash
sudo systemctl restart fastnetmon
sudo systemctl enable fastnetmon
sudo systemctl status fastnetmon
```

Verify the exporter endpoint locally:

```bash
curl http://127.0.0.1:9209/metrics
```

## 2. Install node_exporter on Ubuntu Server

The Grafana dashboard "Node Exporter Full" requires node_exporter metrics.
If node_exporter is missing, the Prometheus target `node-exporter` will stay DOWN and dashboard panels will show No data.

Create a system user:

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin --system node_exporter
```

Download the latest release:

```bash
VERSION=$(curl -fsSL https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -oP '"tag_name": "\K[^"]+')
wget https://github.com/prometheus/node_exporter/releases/download/${VERSION}/node_exporter-${VERSION#v}.linux-amd64.tar.gz
tar -xzf node_exporter-${VERSION#v}.linux-amd64.tar.gz
sudo cp node_exporter-${VERSION#v}.linux-amd64/node_exporter /usr/local/bin/
```

Create the systemd service:

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

Start and enable node_exporter:

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

Open the firewall if needed:

```bash
sudo ufw allow 9100/tcp
```

Verify the local endpoint:

```bash
curl http://127.0.0.1:9100/metrics
sudo ss -lntp | grep 9100
```

If this fails, Prometheus will not be able to scrape the target.

## 3. Prometheus Configuration

Use this scrape config in `prometheus.yml`:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'ubuntu-server'
    metrics_path: /metrics
    fallback_scrape_protocol: PrometheusText0.0.4
    static_configs:
      - targets: ['<YOUR IP>:9209']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['<YOUR IP>:9100']
```

Notes:

- `ubuntu-server` is the FastNetMon metrics job.
- `node-exporter` is required for the standard Node Exporter dashboard.
- If you change the IP, update both targets.

## 4. Docker Compose

Run Prometheus and Grafana:

```bash
docker-compose up -d
```

Check containers:

```bash
docker ps
```

If Prometheus runs in Docker, it must be able to reach `<YOUR IP>:9100` and `<YOUR IP>:9209` over the network.

## 5. Verification Checklist

In Prometheus UI:

- Open `Status > Target health`
- Confirm `ubuntu-server` is UP
- Confirm `node-exporter` is UP

Endpoints to test directly:

- `http://<YOUR IP>:9209/metrics`
- `http://<YOUR IP>:9100/metrics`

If `node-exporter` shows DOWN with connection refused:

- node_exporter service is not running
- node_exporter is bound to another port
- firewall or network access is blocking port 9100

## 6. Grafana Dashboard Behavior

The dashboard "Node Exporter Full" uses node_exporter metrics such as:

- `node_cpu_seconds_total`
- `node_memory_MemAvailable_bytes`
- `node_filesystem_size_bytes`
- `node_network_receive_bytes_total`

FastNetMon metrics on port `9209` are different, so they do not fill the Node Exporter dashboard panels.

## 7. Common Issues

- No data in Grafana panels:
  - Check the selected datasource is Prometheus
  - Check the dashboard variables `Job`, `Nodename`, `Instance`
  - Make sure the `node-exporter` target is UP in Prometheus

- Connection refused on `9100`:
  - node_exporter is missing
  - node_exporter service failed to start
  - firewall blocks port `9100`

- Prometheus target `9209` is UP but Grafana dashboard is empty:
  - The dashboard expects node_exporter metrics, not FastNetMon metrics

## 8. Files in This Project

- `docker-compose.yml` starts Prometheus and Grafana
- `prometheus.yml` contains the scrape targets
- `grafana/provisioning/datasources/prometheus.yml` configures the Prometheus datasource in Grafana

This setup is ready to publish to GitHub as a simple monitoring lab project.