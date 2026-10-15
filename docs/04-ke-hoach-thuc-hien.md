# 4. Kế hoạch thực hiện và đánh giá

## 4.1. Kế hoạch thực hiện và lịch báo cáo

- **Lịch báo cáo định kỳ:** Thứ Năm (Thu), khung giờ **20:30 – 22:00**.
- **Các tuần báo cáo mốc (Milestone):** **W7, W10, W13, W16, W19**.

### 4.1.1. Các mốc báo cáo chính

| Mốc báo cáo | Thời gian | Giai đoạn | Nội dung trọng tâm | Sản phẩm / Demo báo cáo |
|---|---|---|---|---|
| **Báo cáo đợt 1** | **Thu, W7** (20:30–22:00) | Tuần 1 – 7 | Chốt đề cương, kiến trúc và quyết định kỹ thuật; dựng lab k3s; triển khai Snort 3, Zeek, Fluent Bit và viết bộ rule chữ ký cơ bản | Tài liệu thiết kế, môi trường lab hoạt động, demo Snort 3 + Zeek xuất JSON log |
| **Báo cáo đợt 2** | **Thu, W10** (20:30–22:00) | Tuần 8 – 10 | Dựng Kafka, Elasticsearch, Grafana cơ bản; thu thập baseline lưu lượng bình thường; cài đặt thuật toán cửa sổ trượt (Threshold) và độ lệch thống kê (Z-score) | Pipeline thu thập log qua Kafka, bộ tham số baseline và kết quả kiểm thử thuật toán thống kê |
| **Báo cáo đợt 3** | **Thu, W13** (20:30–22:00) | Tuần 11 – 13 | Xây dựng Anomaly Detection Service (FastAPI) xử lý flow realtime từ Kafka; xây dựng Correlation Engine tính `risk_score`; hoàn thiện dashboard Grafana và Alertmanager | Hệ thống IDS kết hợp chạy end-to-end, demo tạo incident có giải thích và gửi cảnh báo |
| **Báo cáo đợt 4** | **Thu, W16** (20:30–22:00) | Tuần 14 – 16 | Đóng gói Docker + Helm, thiết lập GitLab CI/CD (test rule/thuật toán với `tcpreplay`, Trivy) và ArgoCD; chạy kịch bản tấn công lab và đo đạc 3 cấu hình | Pipeline CI/CD + GitOps, bộ dữ liệu thực nghiệm lab, bảng số liệu so sánh 3 cấu hình |
| **Báo cáo đợt 5** | **Thu, W19** (20:30–22:00) | Tuần 17 – 19 | Phân tích kết quả thực nghiệm, tinh chỉnh ngưỡng phát hiện, hoàn thiện quyển báo cáo, slide thuyết trình và kịch bản demo bảo vệ | Hồ sơ đồ án hoàn chỉnh, slide bảo vệ và hệ thống demo tổng duyệt |

### 4.1.2. Kế hoạch công việc hàng tuần

| Tuần | Nội dung công việc hàng tuần | Đầu ra |
|---|---|---|
| **W1 – W2** | Khảo sát bài toán, nghiên cứu Snort 3, Zeek và các thuật toán phát hiện xâm nhập dựa trên thống kê; chốt đề cương, đặc tả yêu cầu và quyết định kỹ thuật | Tài liệu `01` đến `06` |
| **W3 – W4** | Dựng môi trường lab trên Kubernetes (k3s), thiết lập máy tạo lưu lượng, máy mục tiêu và cấu hình thu thập gói tin | Hạ tầng và sơ đồ mạng lab |
| **W5 – W6** | Triển khai Snort 3 và Zeek trên lab; viết rule phát hiện quét cổng, ICMP sweep, SSH brute force, web attack; cấu hình Fluent Bit chuẩn hóa JSON | Sensor Snort 3 + Zeek và bộ rule cơ bản |
| **W7** | Kiểm thử luồng bắt gói và sinh log; tổng hợp kết quả giai đoạn 1 và **báo cáo tiến độ W7 (Thu, 20:30–22:00)** | Báo cáo và demo mốc W7 |
| **W8** | Triển khai Kafka làm event bus, kết nối Elasticsearch và dựng dashboard Grafana cơ bản để quan sát alert/flow | Pipeline Fluent Bit → Kafka → Elasticsearch |
| **W9** | Thu thập lưu lượng bình thường trong lab để xây dựng baseline; trích xuất đặc trưng flow theo cửa sổ thời gian trượt (60 giây) | Bộ đặc trưng flow và thông số baseline (`mean`, `std`) |
| **W10** | Cài đặt thuật toán luật ngưỡng tần suất (Lớp 1) và phát hiện bất thường Z-score (Lớp 2) trên dữ liệu flow; **báo cáo tiến độ W10 (Thu, 20:30–22:00)** | Module thuật toán phát hiện bất thường và báo cáo mốc W10 |
| **W11** | Đóng gói Anomaly Detection Service bằng Python + FastAPI, tiêu thụ flow từ Kafka và tính `threshold_score`, `anomaly_score` thời gian thực | Dịch vụ phát hiện bất thường realtime |
| **W12** | Xây dựng Correlation Engine hợp nhất cảnh báo Snort và kết quả thống kê trong cửa sổ 60 giây, tính `risk_score` và sinh thông tin giải thích (`evidence`) | Module tương quan và giải thích cảnh báo |
| **W13** | Thiết kế dashboard Grafana hiển thị incident giải thích được, cấu hình Prometheus + Alertmanager; **báo cáo tiến độ W13 (Thu, 20:30–22:00)** | Hệ thống IDS kết hợp hoàn chỉnh và báo cáo mốc W13 |
| **W14** | Đóng gói bằng Docker + Helm; cấu hình GitLab CI/CD (lint, replay PCAP bằng `tcpreplay`, quét Trivy), ArgoCD, RBAC và NetworkPolicy | Quy trình CI/CD và GitOps |
| **W15** | Phát sinh lưu lượng bình thường và chạy các kịch bản tấn công trong mạng lab (Nmap, ICMP sweep, SSH brute force, web attack, DoS giới hạn) | Dữ liệu thực nghiệm trong lab |
| **W16** | Đo đạc chỉ số phát hiện (Precision, Recall, F1, FPR) và hiệu năng (độ trễ, CPU/RAM, throughput) trên 3 cấu hình Signature-only, Anomaly-only, Hybrid; **báo cáo tiến độ W16 (Thu, 20:30–22:00)** | Bộ số liệu đánh giá và báo cáo mốc W16 |
| **W17** | Phân tích, đối chiếu số liệu thực nghiệm giữa 3 cấu hình; đánh giá các trường hợp cảnh báo giả và tinh chỉnh bộ ngưỡng | Chương kết quả thực nghiệm và đánh giá |
| **W18** | Hoàn thiện toàn bộ quyển báo cáo đồ án, tài liệu hướng dẫn cài đặt – vận hành và kịch bản demo | Bản thảo báo cáo đồ án hoàn chỉnh |
| **W19** | Hoàn thiện slide thuyết trình, chạy thử demo tổng duyệt và **báo cáo đợt cuối W19 (Thu, 20:30–22:00)** | Hồ sơ bảo vệ đồ án |

## 4.2. Kịch bản kiểm thử dự kiến

- Quét cổng (Port scan) trong mạng lab.
- ICMP sweep trên dải IP thử nghiệm.
- Nhiều lần đăng nhập SSH thất bại (Brute force) vào máy được cấp phép.
- Truy vấn web chứa mẫu bất thường (SQLi, XSS, Path Traversal) trên máy chủ thử nghiệm.
- Gửi lưu lượng tần suất cao (DoS mô phỏng giới hạn) trong mạng lab.
- Lưu lượng hợp lệ tăng đột biến để kiểm tra cảnh báo giả (False Positive).
- Replay PCAP bằng `tcpreplay` để kiểm thử rule và thuật toán tự động.

## 4.3. Chỉ số đánh giá

- True Positive Rate / Recall.
- Precision.
- False Positive Rate.
- F1-score.
- Thời gian từ lúc phát sinh sự kiện đến lúc hiển thị cảnh báo (Detection Latency).
- Mức sử dụng CPU, RAM và dung lượng lưu trữ.
- Throughput Mbps/pps và tỷ lệ rớt gói của Snort.

## 4.4. Tiêu chí hoàn thành

Hệ thống phải nhận được cảnh báo từ Snort 3 và dữ liệu flow từ Zeek, phân tích được hành vi bất thường bằng thuật toán thống kê/luật ngưỡng, lưu đúng schema, hiển thị được trên giao diện, phát hiện được các kịch bản chính và có kết quả đánh giá lặp lại được trong môi trường lab.

## 4.5. Chiến lược so sánh

1. **Signature-only (Snort-only):** chỉ sử dụng cảnh báo dựa trên chữ ký từ Snort 3.
2. **Anomaly-only (Statistical/Threshold-only):** chỉ sử dụng kết quả của thuật toán luật ngưỡng và Z-score trên flow mạng.
3. **Hybrid (Snort + Statistical Correlation):** hợp nhất cảnh báo Snort và thuật toán thống kê bằng Correlation Engine.

Luận điểm cần kiểm chứng là cấu hình kết hợp (Hybrid) có thể tăng khả năng phát hiện các hành vi chưa khớp rule tĩnh hoặc giảm cảnh báo giả so với từng phương pháp riêng lẻ, trong điều kiện cùng dữ liệu và cùng kịch bản thử nghiệm.

## 4.6. Nguồn dữ liệu

Dữ liệu phục vụ cấu hình ngưỡng (baseline) và đánh giá được thu thập trực tiếp từ môi trường mạng lab được kiểm soát (gồm lưu lượng bình thường như DNS, HTTP/HTTPS, SSH hợp lệ và lưu lượng tấn công thử nghiệm), kết hợp các file PCAP chuẩn hóa để replay lặp lại qua `tcpreplay`.
