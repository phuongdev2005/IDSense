# 3. Thiết kế kiến trúc

## 3.1. Kiến trúc logic

```text
Lưu lượng mạng
      |
      v
  Traffic mirror / TAP / SPAN
             |
      +------+------+
      v             v
  Snort 3       Flow extractor
 signature       flow features
      |             |
      +------v------+
          Fluent Bit
              |
              Kafka
          /         \
         v           v
  lưu trữ log    Anomaly Detection Service
                 (Threshold + Z-score)
                      |
                      v
              Correlation + risk score
                      |
       +--------------+--------------+
       v                             v
  Elasticsearch                 Alertmanager
       |                    Telegram/Email/Slack
       v
    Grafana
```

## 3.2. Các module

### Cảm biến Snort

Phân tích gói tin và phát hiện chữ ký theo bộ luật. Snort 3 dự kiến chạy dạng DaemonSet với `hostNetwork` hoặc nhận traffic mirror. Kết quả xuất bằng `alert_json`.

### Flow extractor

Trích xuất đặc trưng kết nối/flow bằng Zeek. Đây là nguồn dữ liệu quan trọng để theo dõi hành vi mạng ở mức luồng kết nối, kể cả khi payload được mã hóa.

### Bộ tiếp nhận và chuẩn hóa

Kiểm tra dữ liệu đầu vào, chuyển các trường về schema thống nhất, loại bỏ bản ghi lỗi và gắn thời gian tiếp nhận.

### Module phát hiện bất thường (Anomaly Detection Service)

Tổng hợp đặc trưng từ flow và tần suất sự kiện theo cửa sổ thời gian trượt (Sliding Window), sau đó áp dụng các thuật toán phát hiện xâm nhập không dùng AI/ML gồm:

- **Luật ngưỡng tần suất (Threshold / Rate-based):** phát hiện quét cổng, ICMP sweep, brute force và flood/DoS dựa trên ngưỡng cấu hình.
- **Độ lệch chuẩn thống kê (Z-score Anomaly Detection):** đo mức độ lệch của lưu lượng hiện tại so với đường cơ sở (baseline) lưu lượng bình thường.

### Correlation engine

Hợp nhất alert từ Snort và kết quả từ Anomaly Detection Service theo cửa sổ thời gian, IP nguồn, IP đích hoặc luồng tấn công; tính điểm rủi ro (`risk_score`) và giảm cảnh báo trùng lặp.

### Kafka và Fluent Bit

Fluent Bit thu nhận log JSON từ Snort và Zeek, Kafka làm lớp đệm và phân phối sự kiện đến Anomaly Detection Service cùng hệ thống lưu trữ.

### Cơ sở dữ liệu

Elasticsearch lưu sự kiện, flow, kết quả phân tích tương quan (`incident`) và phục vụ tìm kiếm, truy vấn lịch sử.

### API và giao diện

Cung cấp truy vấn sự kiện, thống kê, chi tiết cảnh báo, bằng chứng giải thích (`evidence`) và các bộ lọc trên Grafana cho người giám sát.

### Kubernetes và DevOps

Snort, Zeek và các service chạy bằng Helm trên Kubernetes (k3s). GitLab CI/CD thực hiện lint, test rule Snort, test thuật toán phát hiện, build image, scan bằng Trivy và triển khai qua ArgoCD theo GitOps.

## 3.3. Luồng xử lý

1. Snort 3 quan sát gói tin và tạo cảnh báo khi khớp luật chữ ký; đồng thời Zeek trích xuất thông tin flow.
2. Fluent Bit thu thập, chuẩn hóa log JSON và đẩy vào Kafka.
3. Anomaly Detection Service đọc dữ liệu từ Kafka, tổng hợp theo cửa sổ thời gian và chạy thuật toán luật ngưỡng + Z-score.
4. Correlation Engine hợp nhất cảnh báo chữ ký và điểm bất thường thống kê thành `incident` kèm `risk_score`.
5. Elasticsearch lưu trữ sự kiện; Grafana hiển thị dashboard và Alertmanager gửi thông báo khi vượt ngưỡng rủi ro.
6. Người giám sát điều tra nguyên nhân qua các chỉ số giải thích và xử lý cảnh báo.

## 3.4. Kiến trúc triển khai

- **Sensor layer:** Snort 3 và Zeek.
- **Streaming layer:** Fluent Bit và Kafka.
- **Detection layer:** Anomaly Detection Service (Threshold + Z-score) và Correlation Engine.
- **Storage layer:** Elasticsearch lưu log/incident; Prometheus lưu metrics hệ thống.
- **Presentation layer:** Grafana và Alertmanager.
- **Delivery layer:** GitLab CI/CD, Container Registry, Helm và ArgoCD.

Các thành phần bảo mật tối thiểu gồm RBAC, NetworkPolicy, Kubernetes Secret và giới hạn quyền của service account.
