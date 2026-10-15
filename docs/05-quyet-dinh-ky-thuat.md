# 5. Quyết định kỹ thuật cố định

Tài liệu này là baseline chính thức của đồ án. Sau khi chốt, không mở rộng thêm công nghệ nếu không có lý do học thuật rõ ràng.

## 5.1. Tên hướng triển khai

**Explainable Hybrid Network IDS with Snort and Machine Learning**

Hệ thống kết hợp phát hiện dựa trên chữ ký của Snort với phát hiện bất thường trên flow mạng. Kết quả cuối cùng có risk score và lý do giải thích được.

## 5.2. Stack cố định

| Lớp | Công nghệ chốt | Vai trò |
|---|---|---|
| Packet sensor | Snort 3 | Phát hiện tấn công đã biết bằng rule |
| Flow extractor | Zeek | Tạo connection/flow metadata dễ phân tích |
| Event bus | Kafka | Đệm và phân phối event |
| ML service | Python + FastAPI | Nhận flow, trả prediction và anomaly score |
| Lưu trữ | Elasticsearch | Lưu alert, flow và incident |
| Dashboard | Grafana | Trực quan hóa và điều tra |
| Alerting | Prometheus + Alertmanager | Theo dõi service và gửi cảnh báo |
| Runtime | Kubernetes trên k3s | Môi trường triển khai lab |
| Packaging | Docker + Helm | Đóng gói và triển khai |
| CI/CD | GitLab CI/CD | Test, build, scan, deploy |
| GitOps | ArgoCD | Đồng bộ manifest lên Kubernetes |

Không dùng đồng thời Loki và Elasticsearch trong phiên bản chính. Không dùng CICFlowMeter làm flow extractor chính vì Zeek cung cấp log flow linh hoạt hơn cho pipeline realtime.

## 5.3. Thuật toán cố định

Nguyên tắc triển khai là bắt đầu từ các phương pháp dễ hiểu, dễ kiểm tra và có thể giải thích bằng số liệu. Không triển khai deep learning hoặc mô hình ensemble phức tạp trong phiên bản đầu.

### Lớp 1: Luật ngưỡng thống kê

Đây là lớp đơn giản nhất, dùng để kiểm tra các hành vi rõ ràng:

- Một source IP truy cập quá nhiều port trong 60 giây → nghi ngờ port scan.
- Quá nhiều lần đăng nhập SSH thất bại trong 60 giây → nghi ngờ brute force.
- Số packet hoặc byte vượt ngưỡng bình thường → nghi ngờ flood/DoS.

Các ngưỡng phải được ghi trong cấu hình và điều chỉnh bằng dữ liệu lab, không chôn cứng trong code.

### Lớp 2: Z-score anomaly detection

Với các đặc trưng như số kết nối, số byte và số port đích, tính:

```text
z = (x - mean_normal) / standard_deviation_normal
```

Nếu `abs(z) >= 3`, flow được đánh dấu bất thường. Phương pháp này dễ giải thích: giá trị đang lệch bao nhiêu độ lệch chuẩn so với traffic bình thường.

### Mô hình học máy chính: Decision Tree nhỏ

Decision Tree dùng để phân loại các flow thành:

- `BENIGN`
- `PORT_SCAN`
- `BRUTE_FORCE`
- `DOS`
- `WEB_ATTACK`

Giới hạn cố định của cây:

- Độ sâu tối đa nhỏ, ví dụ `max_depth=5`.
- Có thể vẽ và đọc từng nhánh quyết định.
- Phù hợp dữ liệu dạng bảng.
- Không cần chuẩn hóa phức tạp.
- Dễ huấn luyện, đóng gói và chạy realtime.

### Mô hình đối chứng: Logistic Regression

Logistic Regression dùng làm baseline tuyến tính để so sánh với Decision Tree.

### Không triển khai ngay từ đầu

Không đưa Random Forest, XGBoost, SVM, Isolation Forest, Autoencoder, LSTM/GRU vào giai đoạn đầu. Các mô hình này chỉ được thảo luận ở phần hướng phát triển hoặc thử nghiệm phụ nếu phiên bản đơn giản đã ổn định.

## 5.4. Dữ liệu cố định

### Giai đoạn huấn luyện

Sử dụng CIC-IDS2017 làm dataset chính vì có traffic bình thường, nhiều loại tấn công và flow features phổ biến.

Sử dụng một phần NSL-KDD chỉ để kiểm tra khả năng chạy nhanh pipeline; không dùng kết quả NSL-KDD làm kết quả chính.

### Giai đoạn kiểm chứng thực tế

Tự tạo dữ liệu lab từ các máy được cấp phép:

- Traffic bình thường: DNS, HTTP/HTTPS, SSH hợp lệ.
- Port scan: Nmap.
- SSH brute force: nhiều lần đăng nhập thất bại.
- Web attack: request thử nghiệm trên DVWA hoặc Juice Shop.
- DoS mô phỏng ở mức giới hạn trong lab.

Dataset chính thức để báo cáo phải tách thành train, validation và test theo thời gian hoặc theo file capture, tránh trộn các flow gần giống nhau giữa train và test.

## 5.5. Đặc trưng đầu vào

Các đặc trưng chính lấy từ Zeek connection log và dữ liệu tổng hợp:

- Thời lượng kết nối.
- Giao thức và dịch vụ.
- Cổng nguồn, cổng đích.
- Số byte và số packet mỗi chiều.
- Tốc độ byte/packet.
- Số kết nối mới từ một source IP trong cửa sổ thời gian.
- Tỷ lệ kết nối thất bại.
- Số lượng port đích khác nhau.
- TCP flags.

Không đưa `src_ip` và `dst_ip` trực tiếp vào model chính để tránh model học thuộc địa chỉ thay vì học hành vi.

## 5.6. Cách hợp nhất Snort và ML

Mỗi event sau khi xử lý có ba thành phần:

```text
snort_score       điểm từ severity/rule của Snort
ml_score          xác suất lớp tấn công của Decision Tree
anomaly_score     điểm bất thường chuẩn hóa từ Z-score
```

Risk score cố định:

```text
risk_score = 0.45 * snort_score
           + 0.35 * ml_score
           + 0.20 * anomaly_score
```

Nếu Snort và ML cùng xác định cùng một source IP, destination IP và khoảng thời gian 60 giây, hệ thống tạo một incident duy nhất thay vì hai alert riêng.

## 5.7. Giải thích cảnh báo

Mỗi incident phải trả lời được:

1. Ai là nguồn phát sinh?
2. Mục tiêu là ai?
3. Snort khớp rule nào?
4. ML dự đoán lớp nào?
5. Đặc trưng nào đóng góp nhiều nhất?
6. Vì sao risk score ở mức hiện tại?

Ví dụ:

> Incident mức cao vì source IP tạo nhiều kết nối đến các port khác nhau trong thời gian ngắn, đồng thời khớp rule port scan của Snort. Các đặc trưng quan trọng là `unique_destination_ports`, `syn_count` và `connection_rate`.

## 5.8. Tiêu chí đánh giá cố định

### Chất lượng phát hiện

- Precision
- Recall
- F1-score
- False Positive Rate
- Confusion matrix
- ROC-AUC nếu phù hợp

### Hiệu năng hệ thống

- Độ trễ từ event đến alert.
- Số event xử lý mỗi giây.
- CPU/RAM của Snort và ML service.
- Tỷ lệ mất event.

### Ba cấu hình bắt buộc so sánh

1. Snort-only.
2. ML-only.
3. Hybrid Snort + ML.

## 5.9. Kết quả cần chứng minh

Đồ án cần kiểm chứng hai luận điểm:

1. Luật ngưỡng và Decision Tree có thể phát hiện các flow bất thường mà Snort chưa có rule.
2. Hybrid giúp tăng khả năng phát hiện và/hoặc giảm cảnh báo giả so với từng phương pháp riêng lẻ.

Nếu kết quả hybrid không tốt hơn ở mọi chỉ số, báo cáo phải phân tích nguyên nhân thay vì điều chỉnh số liệu.
