run docker-compose up -d

----------------------------------tải FastNetMon trong ubuntu server 
- update và nâng cấp hệ thống
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y wget curl
- tải FastNetMon
    wget https://install.fastnetmon.com/installer -O installer
    sudo chmod +x installer
    sudo ./installer -install_community_edition 
- nhap thong gmail doanh nghiep de nhan key FastNetMon
- cau hinh FastNetMon
    sudo nano /etc/fastnetmon.conf
    # thay đổi các thông số va khong luu IP auto ban trong file /etc/fastnetmon.conf chinh sua nhu sau:
    enable_ban = off

    prometheus = on
    prometheus_port = 9209
    prometheus_host = 0.0.0.0

    process_ban = off






----------------------------------tải prometheus va grafana bang docker
- tao 2 file docker-compose.yml va prometheus.yml
- khoi dong lai he thong : docker-compose up -d --force-recreate 
- tao file prometheus.yml trong folder grafana/provisioning/datasources/prometheus.yml có chức năng kết nối Prometheus với Grafana
