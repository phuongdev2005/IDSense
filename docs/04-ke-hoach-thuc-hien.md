# 4. Kế hoạch thực hiện và đánh giá

## 4.1. Lộ trình 12 tháng

| Giai đoạn | Nội dung | Sản phẩm |
|---|---|---|
| Tháng 1–2 | Học Snort 3, Kubernetes, GitLab CI; dựng lab | Nền tảng và sơ đồ lab |
| Tháng 3–4 | Snort trên Kubernetes, JSON log, dashboard cơ bản, rule tự viết | Sensor và rule |
| Tháng 5–7 | Tiền xử lý dataset, chọn đặc trưng, huấn luyện và so sánh model | Kết quả ML offline |
| Tháng 8–9 | Tích hợp ML realtime, Kafka và correlation alert | Pipeline hybrid |
| Tháng 10 | CI/CD, GitOps, Trivy, RBAC, NetworkPolicy, observability | Quy trình triển khai |
| Tháng 11 | Chạy kịch bản tấn công lab, đo đạc, so sánh ba cấu hình | Bộ số liệu đánh giá |
| Tháng 12 | Hoàn thiện báo cáo, slide và chuẩn bị bảo vệ | Hồ sơ đồ án |

## 4.2. Kịch bản kiểm thử dự kiến

- Quét cổng trong mạng lab.
- ICMP sweep trên dải IP thử nghiệm.
- Nhiều lần đăng nhập SSH thất bại vào máy được cấp phép.
- Truy vấn web chứa mẫu bất thường trong máy chủ thử nghiệm.
- Lưu lượng hợp lệ tăng đột biến để kiểm tra cảnh báo giả.
- Replay PCAP bằng tcpreplay để kiểm thử rule tự động.

## 4.3. Chỉ số đánh giá

- True Positive Rate / Recall.
- Precision.
- False Positive Rate.
- F1-score.
- Thời gian từ lúc phát sinh sự kiện đến lúc hiển thị cảnh báo.
- Mức sử dụng CPU, RAM và dung lượng lưu trữ.
- Throughput Mbps/pps và tỷ lệ rớt gói của Snort.

## 4.4. Tiêu chí hoàn thành

Hệ thống phải nhận được cảnh báo từ Snort, lưu đúng schema, hiển thị được trên giao diện, phân loại được các kịch bản chính và có kết quả đánh giá lặp lại được trong môi trường lab.

## 4.5. Chiến lược so sánh

1. **Signature-only:** chỉ sử dụng cảnh báo từ Snort.
2. **ML-only:** chỉ sử dụng kết quả của mô hình.
3. **Hybrid:** hợp nhất Snort và ML bằng correlation engine.

Luận điểm cần kiểm chứng là cấu hình hybrid có thể tăng khả năng phát hiện hoặc giảm false positive so với từng phương pháp riêng lẻ, trong điều kiện cùng dữ liệu và cùng kịch bản thử nghiệm.

## 4.6. Nguồn dữ liệu

Có thể bắt đầu với NSL-KDD để thử nhanh, sau đó mở rộng sang CIC-IDS2017/2018 và UNSW-NB15. Cần tự thu thập thêm traffic từ lab để kiểm chứng khả năng tổng quát hóa; kết quả từ dataset công khai không được xem là đại diện hoàn toàn cho traffic thực tế.
