# Thiết lập stack giám sát

Dự án này chạy:

- Chỉ số exporter của FastNetMon trên cổng `9209`
- node_exporter trên cổng `9100`
- Prometheus để thu thập dữ liệu từ cả hai mục tiêu
- Grafana để trực quan hóa số liệu

## Khởi động nhanh prometheus và grafana bằng Docker Compose

Tạo các file `docker-compose.yml` và `prometheus.yml`, sau đó chạy:

```bash
docker-compose up -d
```

## 1. Cài đặt FastNetMon trên Ubuntu Server

Cập nhật gói phần mềm và cài đặt các công cụ hỗ trợ:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl
```

Tải về và cài đặt FastNetMon Community Edition:

```bash
wget https://install.fastnetmon.com/installer -O installer
sudo chmod +x installer
sudo ./installer -install_community_edition
```

Hoàn tất quá trình thiết lập FastNetMon trong trình cài đặt nếu nó hỏi về email hoặc khóa bản quyền.

Chỉnh sửa cấu hình FastNetMon:

```bash
sudo nano /etc/fastnetmon.conf
```

Cấu hình Prometheus khuyến nghị:

```ini
enable_ban = on
prometheus = on
prometheus_port = 9209
prometheus_host = 0.0.0.0
```

Khởi động lại và bật dịch vụ:

```bash
sudo systemctl restart fastnetmon
sudo systemctl enable fastnetmon
sudo systemctl status fastnetmon
```

Xác minh endpoint exporter trên máy cục bộ:

```bash
curl http://127.0.0.1:9209/metrics
```

## 2. Cài đặt node_exporter trên Ubuntu Server

Bảng điều khiển Grafana "Node Exporter Full" yêu cầu các số liệu từ node_exporter.
Nếu node_exporter chưa được cài đặt, mục tiêu Prometheus `node-exporter` sẽ ở trạng thái DOWN và các panel trên dashboard sẽ hiển thị "No data".

Tạo người dùng hệ thống:

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin --system node_exporter
```

Tải bản phát hành mới nhất:

```bash
VERSION=$(curl -fsSL https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -oP '"tag_name": "\K[^"]+')
wget https://github.com/prometheus/node_exporter/releases/download/${VERSION}/node_exporter-${VERSION#v}.linux-amd64.tar.gz
tar -xzf node_exporter-${VERSION#v}.linux-amd64.tar.gz
sudo cp node_exporter-${VERSION#v}.linux-amd64/node_exporter /usr/local/bin/
```

Tạo dịch vụ systemd:

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

Khởi động và bật node_exporter:

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

Xác minh endpoint cục bộ:

```bash
curl http://127.0.0.1:9100/metrics
sudo ss -lntp | grep 9100
```

Nếu bước này thất bại, Prometheus sẽ không thể scrape mục tiêu này.

## 3. Cấu hình Prometheus

Sử dụng cấu hình scrape này trong `prometheus.yml`:

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

Ghi chú:

- `ubuntu-server` là job thu thập metrics của FastNetMon.
- `node-exporter` là bắt buộc để dùng bảng điều khiển Node Exporter chuẩn.
- Nếu thay đổi IP, hãy cập nhật cả hai target.

## 4. Docker Compose

Chạy Prometheus và Grafana:

```bash
docker-compose up -d
```

Kiểm tra container:

```bash
docker ps
```

Nếu Prometheus chạy trong Docker, nó phải có thể kết nối tới `<YOUR IP>:9100` và `<YOUR IP>:9209` qua mạng.

## 5. Checklist xác minh

Trong giao diện Prometheus:

- Mở `Status > Target health`
- Xác nhận `ubuntu-server` đang ở trạng thái UP
- Xác nhận `node-exporter` đang ở trạng thái UP

Các endpoint cần kiểm tra trực tiếp:

- `http://<YOUR IP>:9209/metrics`
- `http://<YOUR IP>:9100/metrics`

Nếu `node-exporter` hiển thị DOWN kèm lỗi connection refused:

- dịch vụ node_exporter chưa chạy
- node_exporter đang bind vào một cổng khác
- firewall hoặc mạng đang chặn cổng 9100

## 6. Cách hoạt động của dashboard Grafana

Dashboard "Node Exporter Full" sử dụng các metric của node_exporter như:

- `node_cpu_seconds_total`
- `node_memory_MemAvailable_bytes`
- `node_filesystem_size_bytes`
- `node_network_receive_bytes_total`

Các metric của FastNetMon trên cổng `9209` khác biệt, nên không lấp đầy các panel của dashboard Node Exporter.

## 7. Ý nghĩa các metric của FastNetMon

Mức cảnh báo khuyến nghị cho giám sát băng thông DDoS:

- 100 Gbps: cảnh báo sớm
- 500 Gbps: mức nghiêm trọng cao
- 1 Tbps: sự cố critical

Hồ sơ demo cho buổi thuyết trình lớp học:

```ini
enable_ban = on
ban_time = 30

ban_for_pps = on
ban_for_bandwidth = on
ban_for_flows = on

threshold_pps = 2000
threshold_mbps = 50
threshold_flows = 2000

threshold_tcp_mbps = 50
threshold_udp_mbps = 50
threshold_icmp_mbps = 50

threshold_tcp_pps = 2000
threshold_udp_pps = 2000
threshold_icmp_pps = 20000
```

Hồ sơ này khá aggressive nên demo sẽ kích hoạt nhanh.

Để tránh chặn IP quản trị `192.168.25.129` trong demo, hãy đưa IP này vào whitelist và giữ nó ra khỏi danh sách mạng mục tiêu bị giám sát:

```ini
white_list_path = /etc/networks_whitelist
networks_list_path = /etc/networks_list
monitor_local_ip_addresses = on
```

Ví dụ nội dung file `/etc/networks_whitelist`:

```text
192.168.25.129/32
```

Hãy chắc chắn rằng chỉ có subnet mục tiêu bị giám sát, còn nếu `192.168.25.129` là máy Kali hoặc host quản trị thì nó không nên nằm trong `networks_list`.

## 8. Cau hinh thong bao qua telegram

file cau hinh thong bao qua telegram: `/usr/local/bin/notify_attack.sh`

```ini
#!/bin/bash

BOT_TOKEN="your_bot_token_here"
CHAT_ID="your_chat_id_here"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] 🚀 FastNetMon monitor đang chạy..."

tail -n0 -F /var/log/fastnetmon.log | grep --line-buffered "We have detected attack" | while read line; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    VICTIM_IP=$(echo "$line" | grep -oP '\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b' | head -1)
    ATTACK_TYPE=$(echo "$line" | grep -oP '(?<=attack type: )\S+' || echo "Unknown")

    # ── Luồng 1: Hiển thị trên màn hình server ──
    echo "⚠️  [$TIMESTAMP] CẢNH BÁO DDoS!"
    echo "    IP nạn nhân : ${VICTIM_IP:-Không xác định}"
    echo "    Loại tấn công: ${ATTACK_TYPE}"
    echo "    Log gốc     : $line"
    echo "────────────────────────────────────────"

sleep 1

REPORT=$(ls -t /var/log/fastnetmon_attacks/*.txt 2>/dev/null | head -1)

PROTOCOL=$(grep "^Attack protocol:" "$REPORT" | cut -d':' -f2- | xargs)
DIRECTION=$(grep "^Attack direction:" "$REPORT" | cut -d':' -f2- | xargs)
PEAK_PPS=$(grep "^Peak attack power:" "$REPORT" | cut -d':' -f2- | xargs)
BANDWIDTH=$(grep "^Total incoming traffic:" "$REPORT" | cut -d':' -f2- | xargs)

# Lấy packet đầu tiên sau "Attack traffic dump"
FIRST_PACKET=$(awk '
/^Attack traffic dump$/ {
    getline
    while ($0 == "") getline
    print
    exit
}' "$REPORT")

SOURCE_IP=$(echo "$FIRST_PACKET" | awk '{print $3}' | cut -d':' -f1)
DEST_PORT=$(echo "$FIRST_PACKET" | awk '{print $5}' | cut -d':' -f2)
TCP_FLAG=$(echo "$FIRST_PACKET" | awk '{print $9}')

PACKETS=$(grep -c "protocol:" "$REPORT")

REPORT_NAME=$(basename "$REPORT")
TG_MESSAGE="🚨 <b>CẢNH BÁO DDoS - FastNetMon</b>

🕐 <b>Thời gian:</b> ${TIMESTAMP}
🎯 <b>IP bị tấn công:</b> <code>${VICTIM_IP}</code>
⚡ <b>Loại tấn công:</b> ${ATTACK_TYPE}

📈 <b>Attack Summary</b>
• Protocol    : ${PROTOCOL}
• Direction   : ${DIRECTION}
• Peak PPS    : ${PEAK_PPS}
• Bandwidth   : ${BANDWIDTH}

🔍 <b>Traffic Sample</b>
• Source IP   : <code>${SOURCE_IP}</code>
• Dest Port   : ${DEST_PORT}
• TCP Flag    : ${TCP_FLAG}
• Packets     : ${PACKETS}

📁 <b>Report:</b>
<code>${REPORT_NAME}</code>"

    curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${CHAT_ID}" \
        --data-urlencode "text=${TG_MESSAGE}" \
        --data-urlencode "parse_mode=HTML" \
        > /dev/null 2>&1 &

done
```
chạy file này bằng lệnh:

```bash
sudo chmod +x /usr/local/bin/notify_attack.sh
sudo /usr/local/bin/notify_attack.sh &
```
## 9. Các vấn đề thường gặp

- Không có dữ liệu trong các panel Grafana:
  - Kiểm tra datasource đã chọn là Prometheus
  - Kiểm tra các biến dashboard `Job`, `Nodename`, `Instance`
  - Đảm bảo mục tiêu `node-exporter` đang ở trạng thái UP trong Prometheus

- Lỗi connection refused trên cổng `9100`:
  - node_exporter chưa được cài đặt
  - dịch vụ node_exporter chưa khởi động thành công
  - firewall chặn cổng `9100`

- Mục tiêu Prometheus `9209` đang UP nhưng dashboard Grafana vẫn trống:
  - Dashboard cần các metric của node_exporter, không phải của FastNetMon

## 8. Các file trong dự án này

- `docker-compose.yml` khởi động Prometheus và Grafana
- `prometheus.yml` chứa danh sách scrape targets
- `grafana/provisioning/datasources/prometheus.yml` cấu hình datasource Prometheus trong Grafana
