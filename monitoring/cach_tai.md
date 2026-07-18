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

## 7. FastNetMon Metrics Meaning

Use these metrics in Grafana when you want to monitor DDoS traffic directly:

- `fastnetmon_total_traffic_bits`: total traffic rate in bits per second. This is the main signal for bandwidth-based DDoS detection.
- `fastnetmon_total_simple_packets_processed`: number of processed packets per second.
- `fastnetmon_total_ipv4_packets` and `fastnetmon_total_ipv6_packets`: packet rate split by protocol family.
- `fastnetmon_total_number_of_hosts`: how many hosts FastNetMon is tracking.
- `fastnetmon_influxdb_writes_failed` and `fastnetmon_influxdb_writes_total`: health of the write pipeline.
- `fastnetmon_speed_recalculation_time_seconds` and `fastnetmon_speed_recalculation_time_microseconds`: how long FastNetMon needs to recalculate traffic speed.

Suggested alert levels for DDoS bandwidth monitoring:

- 100 Gbps: early warning
- 500 Gbps: high severity
- 1 Tbps: critical incident

Demo profile for class presentation:

```ini
enable_ban = on
ban_time = 10

ban_for_pps = on
ban_for_bandwidth = on
ban_for_flows = on

threshold_pps = 10
threshold_mbps = 1
threshold_flows = 10

threshold_tcp_mbps = 2
threshold_udp_mbps = 2
threshold_icmp_mbps = 1

threshold_tcp_pps = 30
threshold_udp_pps = 30
threshold_icmp_pps = 20
```

This profile is intentionally aggressive so the demo triggers quickly.

To prevent banning the management IP `192.168.25.129` in the demo, put it in the whitelist and keep it out of the monitored victim network list:

```ini
white_list_path = /etc/networks_whitelist
networks_list_path = /etc/networks_list
monitor_local_ip_addresses = on
```

Example `/etc/networks_whitelist` content:

```text
192.168.25.129/32
```

Make sure the victim subnet is the only subnet you monitor for attacks. If `192.168.25.129` is your Kali box or management host, it should not be in `networks_list`.

Where to see top 10 DDoS IPs:

- In the current Prometheus/Grafana setup, the dashboard does not yet have per-attacker IP metrics.
- FastNetMon itself tracks per-host counters, so the top talkers are available in FastNetMon's own host counters/API path, not in the aggregate Prometheus metrics shown by this dashboard.
- If you want top 10 IPs inside Grafana, the exporter must expose per-host/per-subnet metrics with labels such as `host` or `source_ip`.

How to make Grafana show the top 10 attacker IPs for real:

1. Extend FastNetMon's exporter code to emit a per-host metric for IPv4/IPv6 host counters.
2. Use the host IP as a Prometheus label, for example `host="192.168.25.130"`.
3. Point the Grafana panel at that metric with `topk(10, sum by (host) (rate(<per_host_metric>[1m])))`.
4. If the code exports `source_ip` instead of `host`, replace the label in the panel query.

Current codebase note:

- The existing Prometheus endpoint only exposes aggregate totals.
- The per-host counters exist in FastNetMon internals, but not as Prometheus time series yet.
- The Grafana Top 10 panel will stay empty until that exporter work is added.

Important limitation:

- The current Prometheus endpoint exposes only aggregate traffic counters.
- It does not expose attacker IP directly yet.
- To show attacker IP in Grafana, FastNetMon must export per-host or per-subnet metrics with a label such as `source_ip` or `host`.
- Once that exists, a panel can use a `topk(...)` query to show the busiest source IPs.

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