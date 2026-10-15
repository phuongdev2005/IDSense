# 5. Quyết định kỹ thuật cố định

Tài liệu này là baseline chính thức của đồ án. Sau khi chốt, không mở rộng thêm công nghệ nếu không có lý do học thuật rõ ràng. Dự án **không sử dụng AI/ML**, chỉ tập trung vào phát hiện theo chữ ký (Snort 3) kết hợp các thuật toán thống kê, luật ngưỡng và tương quan sự kiện.

## 5.1. Tên hướng triển khai

**Explainable Hybrid Network IDS with Snort and Statistical Anomaly Detection**

Hệ thống kết hợp phát hiện dựa trên chữ ký của Snort 3 với các thuật toán phát hiện bất thường dựa trên thống kê và luật ngưỡng trên flow mạng. Kết quả cuối cùng có `risk_score` và lý do giải thích tường minh bằng số liệu.

## 5.2. Stack cố định

| Lớp | Công nghệ chốt | Vai trò |
|---|---|---|
| Packet sensor | Snort 3 | Phát hiện tấn công đã biết bằng rule chữ ký |
| Flow extractor | Zeek | Tạo connection/flow metadata phục vụ phân tích hành vi |
| Log collector | Fluent Bit | Thu thập và chuẩn hóa log JSON từ Snort và Zeek |
| Event bus | Kafka | Đệm và phân phối event |
| Detection service | Python + FastAPI | Chạy thuật toán cửa sổ trượt (Threshold), Z-score và Correlation |
| Lưu trữ | Elasticsearch | Lưu alert, flow và incident |
| Dashboard | Grafana | Trực quan hóa và điều tra sự cố |
| Alerting | Prometheus + Alertmanager | Theo dõi service và gửi cảnh báo |
| Runtime | Kubernetes trên k3s | Môi trường triển khai lab |
| Packaging | Docker + Helm | Đóng gói và triển khai |
| CI/CD | GitLab CI/CD | Test rule/thuật toán, build, scan, deploy |
| GitOps | ArgoCD | Đồng bộ manifest lên Kubernetes |

Không dùng đồng thời Loki và Elasticsearch trong phiên bản chính. Không dùng CICFlowMeter làm flow extractor chính vì Zeek cung cấp log flow linh hoạt và ổn định hơn cho pipeline realtime.

## 5.3. Thuật toán phát hiện cố định (Không dùng AI/ML)

Nguyên tắc triển khai là sử dụng các thuật toán thống kê và luật ngưỡng tất định, dễ hiểu, dễ kiểm chứng và giải thích trực tiếp bằng số liệu mạng thực tế.

### Lớp 1: Thuật toán cửa sổ trượt và luật ngưỡng tần suất (Sliding-Window Threshold Detection)

Dữ liệu flow từ Zeek được gom theo cửa sổ thời gian trượt `W = 60 giây` cho từng cặp `(src_ip, dst_ip)` hoặc theo `src_ip` để phát hiện các hành vi bất thường rõ ràng:

- **Port Scan (`PORT_SCAN`):** Một `src_ip` kết nối tới số lượng cổng đích khác nhau (`unique_dst_ports >= N_port`) hoặc có tỷ lệ kết nối không hoàn tất (`failed_connection_ratio >= R_fail`, ví dụ nhiều gói `S0`/`REJ`) trong 60 giây.
- **ICMP Sweep (`ICMP_SWEEP`):** Một `src_ip` gửi gói ICMP tới số lượng IP đích khác nhau (`unique_dst_ips >= N_icmp`) trong 60 giây.
- **Brute Force (`BRUTE_FORCE`):** Quá nhiều kết nối lặp lại tới cổng dịch vụ xác thực (như SSH cổng 22, HTTP login) từ cùng một `src_ip` trong 60 giây (`auth_conn_count >= N_auth`).
- **DoS / Flood (`DOS`):** Tốc độ gói tin (`packets_per_window`) hoặc dung lượng byte (`bytes_per_window`) vượt ngưỡng chịu tải bình thường của dịch vụ.

Mức độ vượt ngưỡng được chuẩn hóa thành `threshold_score` trong đoạn `[0, 100]`. Các ngưỡng phải được khai báo trong file cấu hình (YAML/JSON) và hiệu chỉnh bằng dữ liệu lab, không chôn cứng trong code.

### Lớp 2: Thuật toán phát hiện bất thường thống kê Z-score (Statistical Anomaly Detection)

Với mỗi đặc trưng lưu lượng $x$ (như số kết nối/phút, tổng số byte, số port đích duy nhất, tỷ lệ cờ SYN lỗi), hệ thống so sánh với đường cơ sở (baseline) của lưu lượng bình thường đã đo trong lab ($\mu_{\text{normal}}, \sigma_{\text{normal}}$):

```text
z = (x - mean_normal) / standard_deviation_normal
```

- Nếu `abs(z) >= 3.0`, đặc trưng đó được đánh dấu là bất thường thống kê.
- Điểm `anomaly_score` (thang `0–100`) được tính dựa trên giá trị $|z|$ lớn nhất hoặc trung bình có trọng số của nhóm đặc trưng vượt ngưỡng.
- Phương pháp này giải thích được trực tiếp: giá trị hiện tại đang lệch bao nhiêu lần độ lệch chuẩn so với lưu lượng bình thường.

### Lớp 3: Thuật toán tương quan sự kiện (Event Correlation)

Kết hợp cảnh báo chữ ký từ Snort 3 và kết quả từ Lớp 1 + Lớp 2 theo cùng `src_ip`, `dst_ip` trong cửa sổ thời gian 60 giây để nâng độ chính xác và giảm cảnh báo trùng lặp.

## 5.4. Dữ liệu cố định

Vì hệ thống không huấn luyện mô hình AI/ML, dữ liệu của đồ án tập trung vào **dữ liệu thực nghiệm trong mạng lab**:

### Giai đoạn xây dựng đường cơ sở (Baseline)

Thu thập lưu lượng bình thường trong môi trường lab để tính các tham số thống kê (`mean_normal`, `std_normal`) và chọn ngưỡng ban đầu:

- Lưu lượng truy vấn DNS, duyệt web HTTP/HTTPS hợp lệ.
- Phiên làm việc SSH hợp lệ và truyền file định kỳ.

### Giai đoạn kiểm chứng tấn công

Tự phát sinh dữ liệu tấn công từ các máy được cấp phép trong mạng lab:

- **Port scan:** quét SYN, FIN, UDP bằng Nmap.
- **ICMP sweep:** dò quét dải mạng bằng ping/fping/nmap.
- **SSH brute force:** thử đăng nhập thất bại liên tục vào máy chủ thử nghiệm.
- **Web attack:** gửi request chứa mẫu SQLi, XSS, Directory Traversal vào DVWA hoặc Juice Shop.
- **DoS mô phỏng:** tạo lưu lượng tần suất cao có kiểm soát trong lab.
- **Traffic đột biến hợp lệ:** tạo tải hợp lệ tăng cao để kiểm tra tỷ lệ cảnh báo giả (False Positive).

Các kịch bản được lưu lại dưới dạng file PCAP để có thể phát lại (replay) bằng `tcpreplay`, đảm bảo mọi lần đo đạc đều lặp lại được trên cùng điều kiện dữ liệu.

## 5.5. Đặc trưng đầu vào

Các đặc trưng chính lấy từ Zeek connection log và tổng hợp trong cửa sổ trượt 60 giây:

- Thời lượng kết nối (`duration`).
- Giao thức và dịch vụ (`proto`, `service`).
- Cổng nguồn, cổng đích (`src_port`, `dst_port`).
- Số byte và số packet mỗi chiều (`orig_bytes`, `resp_bytes`, `orig_pkts`, `resp_pkts`).
- Tốc độ byte/packet trên giây.
- Số kết nối mới từ một `src_ip` trong cửa sổ 60 giây (`connection_count`).
- Tỷ lệ kết nối thất bại (`failed_connection_ratio` dựa trên `conn_state` như `S0`, `REJ`, `RSTO`).
- Số lượng cổng đích khác nhau (`unique_dst_ports`) và IP đích khác nhau (`unique_dst_ips`).
- Trạng thái kết nối và cờ TCP.

## 5.6. Cách hợp nhất Snort và thuật toán thống kê

Mỗi sự kiện/cửa sổ sau khi xử lý có ba thành phần điểm (thang `0–100`):

```text
snort_score       điểm quy đổi từ priority/severity của rule Snort
threshold_score   điểm mức độ vi phạm luật ngưỡng tần suất (Lớp 1)
anomaly_score     điểm bất thường chuẩn hóa từ độ lệch Z-score (Lớp 2)
```

Công thức tính `risk_score` cố định:

```text
risk_score = 0.45 * snort_score
           + 0.35 * threshold_score
           + 0.20 * anomaly_score
```

Nếu Snort và dịch vụ phát hiện bất thường cùng ghi nhận sự kiện trên cùng `src_ip`, `dst_ip` trong khoảng thời gian 60 giây, hệ thống hợp nhất thành **một incident duy nhất** thay vì sinh nhiều cảnh báo rời rạc.

## 5.7. Giải thích cảnh báo

Mỗi incident phải trả lời rõ ràng 6 câu hỏi:

1. Ai là nguồn phát sinh (`src_ip`)?
2. Mục tiêu là ai (`dst_ip`, `dst_port`, `service`)?
3. Snort khớp rule chữ ký nào (nếu có)?
4. Luật ngưỡng nào bị vi phạm (`PORT_SCAN`, `ICMP_SWEEP`, `BRUTE_FORCE`, `DOS`)?
5. Đặc trưng nào có độ lệch Z-score cao nhất so với baseline?
6. Vì sao `risk_score` ở mức hiện tại?

Ví dụ:

> Incident mức cao (`risk_score = 92.5`) vì `src_ip 10.10.0.5` tạo kết nối tới 42 cổng đích khác nhau trong 60 giây (vượt ngưỡng `unique_dst_ports >= 20`, `z = 6.4`), đồng thời khớp rule port scan của Snort.

## 5.8. Tiêu chí đánh giá cố định

### Chất lượng phát hiện

- Precision
- Recall (True Positive Rate)
- F1-score
- False Positive Rate (FPR)
- Bảng ma trận nhầm lẫn (Confusion Matrix) theo từng kịch bản tấn công

### Hiệu năng hệ thống

- Độ trễ từ lúc phát sinh sự kiện đến lúc tạo incident/alert.
- Số event/flow xử lý mỗi giây (Throughput).
- Mức sử dụng CPU/RAM của Snort 3 và Anomaly Detection Service.
- Tỷ lệ rớt gói của Snort hoặc mất event trên pipeline.

### Ba cấu hình bắt buộc so sánh

1. **Snort-only (Signature-only):** chỉ bật cảnh báo từ Snort 3.
2. **Anomaly-only (Statistical/Threshold-only):** chỉ bật thuật toán cửa sổ trượt + Z-score trên flow Zeek.
3. **Hybrid (Snort + Statistical Correlation):** kết hợp cả hai qua Correlation Engine.

## 5.9. Kết quả cần chứng minh

Đồ án cần kiểm chứng hai luận điểm:

1. Thuật toán luật ngưỡng trượt và Z-score trên flow có thể phát hiện các hành vi quét, vét cạn hoặc flood bất thường khi payload mã hóa hoặc khi Snort chưa có rule đặc thù.
2. Cấu hình kết hợp (Hybrid) giúp tăng khả năng phát hiện và/hoặc giảm cảnh báo giả so với từng phương pháp riêng lẻ.

Nếu kết quả Hybrid không tốt hơn ở mọi chỉ số, báo cáo phải phân tích rõ nguyên nhân kỹ thuật từ dữ liệu thực nghiệm thay vì điều chỉnh số liệu.
