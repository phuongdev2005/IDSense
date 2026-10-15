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
  lưu trữ log    ML Detection Service
                      |
                      v
              Correlation + risk score
                      |
       +--------------+--------------+
       v                             v
  Elasticsearch/Loki             Alertmanager
       |                    Telegram/Email/Slack
       v
  Grafana/Kibana
```

## 3.2. Các module

### Cảm biến Snort

Phân tích gói tin và phát hiện chữ ký theo bộ luật. Snort 3 dự kiến chạy dạng DaemonSet với `hostNetwork` hoặc nhận traffic mirror. Kết quả xuất bằng `alert_json`.

### Flow extractor

Trích xuất đặc trưng flow bằng Zeek, Argus hoặc CICFlowMeter. Đây là nguồn dữ liệu quan trọng trong trường hợp payload được mã hóa.

### Bộ tiếp nhận và chuẩn hóa

Kiểm tra dữ liệu đầu vào, chuyển các trường về schema thống nhất, loại bỏ bản ghi lỗi và gắn thời gian tiếp nhận.

### Module phát hiện bất thường

Tạo đặc trưng từ flow và tần suất sự kiện, sau đó tính điểm bất thường hoặc phân loại. Các thuật toán dự kiến so sánh gồm Random Forest/XGBoost/SVM và Isolation Forest/Autoencoder/One-Class SVM. LSTM/GRU chỉ thực hiện nếu còn đủ thời gian và dữ liệu.

### Correlation engine

Hợp nhất alert từ Snort và ML theo thời gian, IP, flow hoặc campaign; tạo risk score và giảm cảnh báo trùng lặp.

### Kafka và Fluent Bit

Fluent Bit thu nhận log JSON, Kafka làm lớp đệm và phân phối sự kiện đến ML service và hệ thống lưu trữ.

### Cơ sở dữ liệu

Lưu sự kiện, kết quả phân tích, trạng thái xử lý và thông tin cấu hình cần thiết.

### API và giao diện

Cung cấp truy vấn sự kiện, thống kê, chi tiết cảnh báo và các bộ lọc cho người giám sát.

### Kubernetes và DevOps

Snort, flow extractor và các service chạy bằng Helm trên Kubernetes. GitLab CI/CD thực hiện lint, test rule, test model, build image, scan bằng Trivy và triển khai qua ArgoCD theo GitOps. Terraform hoặc Ansible được cân nhắc cho hạ tầng lab.

## 3.3. Luồng xử lý

1. Snort quan sát lưu lượng.
2. Snort tạo cảnh báo khi khớp luật.
3. Collector nhận và chuẩn hóa cảnh báo.
4. Hệ thống lưu sự kiện và chạy phân tích bổ sung.
5. Dashboard hiển thị cảnh báo, điểm rủi ro và thống kê.
6. Người giám sát điều tra và đánh dấu kết quả xử lý.

## 3.3. Kiến trúc triển khai

- **Sensor layer:** Snort DaemonSet và flow extractor.
- **Streaming layer:** Fluent Bit và Kafka.
- **Detection layer:** ML Detection Service và correlation.
- **Storage layer:** Elasticsearch hoặc Loki; Prometheus lưu metrics.
- **Presentation layer:** Grafana/Kibana và Alertmanager.
- **Delivery layer:** GitLab CI/CD, Container Registry, Helm và ArgoCD.

Các thành phần bảo mật tối thiểu gồm RBAC, NetworkPolicy, Kubernetes Secret và giới hạn quyền của service account.
