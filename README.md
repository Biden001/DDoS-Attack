# 🌐 Thực Tập niên luận: Xây Dựng Hệ Thống Giám Sát Mạng Và Cảnh Báo Sớm Tấn Công DDoS

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Server-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Fastnetmon](https://img.shields.io/badge/Core-Fastnetmon-0052CC?style=flat-square&logo=linux&logoColor=white)
![Prometheus](https://img.shields.io/badge/Database-Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Dashboard-Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)
![Telegram](https://img.shields.io/badge/Alert-Telegram%20Bot-26A5E4?style=flat-square&logo=telegram&logoColor=white)

Dự án này là tài liệu và mã nguồn phục vụ cho đồ án thực tập viết niên luận với mục tiêu nghiên cứu, thiết kế và triển khai một hệ thống giám sát dòng lưu lượng tại biên mạng. Hệ thống tích hợp cơ chế cảnh báo hai luồng nhằm tối ưu tốc độ phát hiện sự cố và giảm thiểu thời gian dịch vụ bị gián đoạn do các cuộc tấn công DDoS.

---

## 🚀 Kiến Trúc & Công Nghệ Sử Dụng

Hệ thống được thiết kế theo mô hình hai máy phối hợp (một Ubuntu Server đóng vai trò nạn nhân/giám sát và một máy Windows 11 làm trung tâm hiển thị), kết hợp các giải pháp mã nguồn mở:

* **Fastnetmon:** Đóng vai trò cốt lõi trong việc thu thập lưu lượng thông qua thư viện AF_PACKET và phát hiện tấn công DDoS dựa trên các ngưỡng lưu lượng thiết lập.
* **Prometheus:** Cơ sở dữ liệu chuỗi thời gian, được cấu hình thu thập dữ liệu (5 giây một lần) từ Fastnetmon để lưu trữ các chỉ số mạng.
* **Grafana:** Nền tảng trực quan hóa dữ liệu, cung cấp các Dashboard đồ họa để giám sát tổng thể tình trạng hệ thống theo thời gian thực.
* **Telegram Bot API:** Kênh cảnh báo phụ trợ, nhận tín hiệu bất thường từ Prometheus và gửi tin nhắn cảnh báo trực tiếp đến thiết bị di động của quản trị viên.
* **Kali Linux:** Môi trường được sử dụng để thực thi các lệnh `hping3`, đóng vai trò là máy tấn công để giả lập các kịch bản DDoS.

---

## 🎯 Chức Năng Chính

* **Phân tích gói tin chuyên sâu:** Tiếp nhận và bóc tách các gói tin trực tiếp ở mức hệ điều hành, lấy các thông tin quan trọng (IP, Port, TCP flag) mà không làm suy giảm hiệu năng của máy chủ.
* **Giám sát thời gian thực:** Phân tích và phát hiện tấn công theo thời gian thực (dưới 10 giây) thông qua việc liên tục so sánh các chỉ số thực tế với ngưỡng giới hạn cho phép.
* **Cảnh báo đa kênh:** Vận hành hệ thống cảnh báo hai luồng song song: hiển thị trực tiếp lên màn hình hệ thống (dành cho người đang trực) và gửi tin nhắn qua Telegram (đảm bảo không bỏ sót cảnh báo).

---

## ⚔️ Kịch Bản Tấn Công Thử Nghiệm

Hệ thống đã được đánh giá qua việc giả lập các phương thức tấn công DDoS phổ biến bằng công cụ `hping3`:

1. **Tấn công tầng mạng (Layer 3):** 
   * Sử dụng các kỹ thuật như *ICMP Flood*, *IP Null Attack* và *Smurf Attack* nhằm làm nghẽn băng thông của máy chủ.
2. **Tấn công tầng giao vận (Layer 4):** 
   * Sử dụng *TCP SYN Flood*, *UDP Flood* và *RST/FIN Flood* để làm cạn kiệt tài nguyên hệ thống và các bảng trạng thái kết nối.

---

## 📊 Kết Quả Đánh Giá

* **Thời gian phản ứng:** Hệ thống hoạt động nhạy bén, cảnh báo trên màn hình Ubuntu Server chỉ mất từ 2-5 giây và mất khoảng 5-10 giây để gửi thông báo tới Telegram Bot API.
* **Độ chính xác:** Đạt mức 100% khi phát hiện đúng các loại tấn công giả lập với băng thông và lượng gói tin đủ lớn.

---

## 👥 Thông Tin Tác Giả

* **Thành viên thực hiện:** Đoàn Quang Huy.

---
