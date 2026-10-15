# 1. Đề cương và định hướng

## 1.1. Tên đề tài

**Xây dựng hệ thống giám sát an ninh mạng sử dụng Snort và các thuật toán phát hiện xâm nhập.**

## 1.2. Bối cảnh

Các cuộc tấn công mạng ngày càng đa dạng, khiến việc giám sát lưu lượng và phát hiện sớm hành vi bất thường trở nên cần thiết. Snort có khả năng phát hiện dựa trên chữ ký, trong khi các thuật toán phát hiện bất thường có thể hỗ trợ nhận biết những mẫu chưa được mô tả rõ bằng luật. Đồ án triển khai theo hướng DevSecOps trên Kubernetes, có CI/CD và quan sát tập trung.

## 1.3. Mục tiêu

- Nghiên cứu nguyên lý hoạt động của hệ thống IDS và Snort 3.
- Thiết kế hệ thống thu thập, chuẩn hóa, lưu trữ và hiển thị cảnh báo.
- Xây dựng các luật Snort phát hiện một số hành vi tấn công phổ biến.
- Nghiên cứu và cài đặt các thuật toán phát hiện bất thường dựa trên thống kê, luật ngưỡng trượt (Sliding-window Threshold) và độ lệch chuẩn (Z-score) trên dữ liệu flow mạng.
- Xây dựng cơ chế tương quan sự kiện (Correlation) và chấm điểm rủi ro (Risk Scoring).
- Đánh giá hệ thống trong môi trường lab được kiểm soát, so sánh ba cấu hình: chỉ Snort (Signature-only), chỉ thuật toán thống kê (Anomaly-only) và IDS kết hợp (Hybrid).
- Đánh giá khả năng tự động kiểm thử rule, thuật toán và triển khai bằng GitLab CI/CD.

## 1.4. Phạm vi

Đồ án tập trung vào giám sát mạng IPv4 trong môi trường thử nghiệm, gồm quét cổng, ICMP sweep, SSH brute force, DoS mô phỏng giới hạn và một số hành vi web bất thường. Hệ thống chỉ cảnh báo, không tự động tấn công hoặc chặn lưu lượng.

## 1.5. Kết quả dự kiến

Một hệ thống IDS kết hợp có Snort 3 làm cảm biến chữ ký, Zeek làm flow extractor, pipeline Fluent Bit + Kafka, dịch vụ phát hiện bất thường thống kê (Anomaly Detection Service), bộ tương quan cảnh báo (Correlation Engine), lưu trữ tập trung (Elasticsearch) và dashboard (Grafana). Hệ thống được đóng gói để triển khai trên Kubernetes.
