# 1. Đề cương và định hướng

## 1.1. Tên đề tài

**Xây dựng hệ thống giám sát an ninh mạng sử dụng Snort và các thuật toán phát hiện xâm nhập.**

## 1.2. Bối cảnh

Các cuộc tấn công mạng ngày càng đa dạng, khiến việc giám sát lưu lượng và phát hiện sớm hành vi bất thường trở nên cần thiết. Snort có khả năng phát hiện dựa trên chữ ký, trong khi các thuật toán phát hiện bất thường có thể hỗ trợ nhận biết những mẫu chưa được mô tả rõ bằng luật. Đồ án triển khai theo hướng DevSecOps trên Kubernetes, có CI/CD và quan sát tập trung.

## 1.3. Mục tiêu

- Nghiên cứu nguyên lý hoạt động của hệ thống IDS và Snort.
- Thiết kế hệ thống thu thập, lưu trữ và hiển thị cảnh báo.
- Xây dựng các luật phát hiện một số hành vi tấn công phổ biến.
- Nghiên cứu một phương pháp phát hiện bất thường trên dữ liệu mạng.
- Đánh giá hệ thống trong môi trường lab được kiểm soát.
- So sánh ba cấu hình: chỉ Snort, chỉ ML và IDS lai.
- Đánh giá khả năng tự động kiểm thử rule, model và triển khai bằng GitLab CI/CD.

## 1.4. Phạm vi

Đồ án tập trung vào giám sát mạng IPv4 trong môi trường thử nghiệm, gồm quét cổng, ICMP sweep, SSH brute force và một số hành vi web bất thường. Hệ thống chỉ cảnh báo, không tự động tấn công hoặc chặn lưu lượng.

## 1.5. Kết quả dự kiến

Một mô hình IDS lai có Snort làm cảm biến signature, flow extractor, pipeline Kafka, dịch vụ ML, correlation alert, lưu trữ tập trung và dashboard. Hệ thống được đóng gói để triển khai trên Kubernetes.
