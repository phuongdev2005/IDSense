# Báo cáo công việc Tuần 2 (W2): Thiết kế kiến trúc, luồng dữ liệu và chốt quyết định kỹ thuật

- **Tuần thực hiện:** Tuần 2 (W2)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 2

1. Thiết kế kiến trúc logic và kiến trúc triển khai của hệ thống giám sát an ninh mạng.
2. Chốt stack công nghệ cố định để giữ phạm vi đồ án tập trung, khả thi và kiểm chứng được.
3. Thiết kế chi tiết 3 lớp thuật toán phát hiện xâm nhập dựa trên thống kê, luật ngưỡng trượt và tương quan sự kiện (không dùng AI/ML).
4. Thiết kế luồng dữ liệu (Data Flow) và kế hoạch thực hiện theo các mốc báo cáo **Thu, W7, W10, W13, W16, W19 (20:30–22:00)**.

---

## 2. Nội dung thực hiện chi tiết

### 2.1. Thiết kế kiến trúc hệ thống 6 lớp

Hệ thống được chia thành 6 lớp tách biệt nhằm đảm bảo tính chịu lỗi và khả năng mở rộng:

1. **Sensor Layer:**
   - **Snort 3:** phân tích gói tin và phát hiện tấn công theo chữ ký (`alert_json`).
   - **Zeek:** trích xuất siêu dữ liệu kết nối (`conn.log`) phục vụ phân tích hành vi ở mức flow.
2. **Streaming Layer:**
   - **Fluent Bit:** thu thập log JSON từ Snort 3 và Zeek, chuẩn hóa trường dữ liệu.
   - **Kafka:** làm hàng đợi sự kiện (event bus) phân phối dữ liệu tới dịch vụ phân tích và kho lưu trữ.
3. **Detection & Correlation Layer:**
   - **Anomaly Detection Service (Python + FastAPI):** gom flow theo cửa sổ trượt 60 giây, tính điểm vượt ngưỡng (`threshold_score`) và điểm lệch chuẩn thống kê (`anomaly_score` từ Z-score).
   - **Correlation Engine:** hợp nhất cảnh báo từ Snort 3 và Anomaly Detection Service trong cửa sổ 60 giây theo `(src_ip, dst_ip)` để tạo `incident` kèm `risk_score`.
4. **Storage Layer:**
   - **Elasticsearch:** lưu trữ toàn bộ alert, flow và incident.
   - **Prometheus:** thu thập metrics hiệu năng của hệ thống.
5. **Presentation & Alerting Layer:**
   - **Grafana:** trực quan hóa dashboard giám sát và điều tra sự cố.
   - **Alertmanager:** gửi thông báo cảnh báo khi `risk_score` vượt ngưỡng.
6. **Delivery Layer:**
   - **Docker + Helm, GitLab CI/CD, Trivy, ArgoCD** triển khai trên cụm **Kubernetes (k3s)**.

### 2.2. Thiết kế thuật toán phát hiện và chấm điểm rủi ro

Hệ thống sử dụng 3 lớp thuật toán tất định, giải thích được hoàn toàn bằng số liệu:

1. **Lớp 1 – Luật ngưỡng tần suất trên cửa sổ trượt 60 giây (`threshold_score`):**
   - Phát hiện `PORT_SCAN` dựa trên `unique_dst_ports` và `failed_connection_ratio`.
   - Phát hiện `ICMP_SWEEP` dựa trên `unique_dst_ips` nhận gói ICMP Echo.
   - Phát hiện `BRUTE_FORCE` dựa trên số kết nối lặp lại tới cổng dịch vụ xác thực (SSH/HTTP).
   - Phát hiện `DOS` dựa trên tốc độ gói tin và dung lượng byte trong cửa sổ 60 giây.
2. **Lớp 2 – Độ lệch chuẩn thống kê Z-score (`anomaly_score`):**
   $$z = \frac{x - \mu_{\text{normal}}}{\sigma_{\text{normal}}}$$
   Đánh dấu bất thường khi $|z| \ge 3.0$ so với đường cơ sở (baseline) của lưu lượng bình thường trong lab.
3. **Lớp 3 – Hợp nhất sự kiện và tính điểm rủi ro (`risk_score`):**
   $$\text{risk\_score} = 0.45 \times \text{snort\_score} + 0.35 \times \text{threshold\_score} + 0.20 \times \text{anomaly\_score}$$

### 2.3. Thiết kế chiến lược đánh giá thực nghiệm

Thiết lập phương pháp so sánh 3 cấu hình độc lập trên cùng tập lưu lượng mạng lab:

1. **Signature-only (Snort-only):** chỉ sử dụng cảnh báo chữ ký từ Snort 3.
2. **Anomaly-only (Statistical/Threshold-only):** chỉ sử dụng thuật toán cửa sổ trượt + Z-score trên flow Zeek.
3. **Hybrid (Snort + Statistical Correlation):** kết hợp cả hai qua Correlation Engine.

---

## 3. Sản phẩm đầu ra tuần 2

- Hoàn thiện [03-thiet-ke-kien-truc.md](../03-thiet-ke-kien-truc.md).
- Hoàn thiện [04-ke-hoach-thuc-hien.md](../04-ke-hoach-thuc-hien.md) theo lịch báo cáo **Thu, W7, W10, W13, W16, W19 (20:30–22:00)**.
- Hoàn thiện [05-quyet-dinh-ky-thuat.md](../05-quyet-dinh-ky-thuat.md) và [06-data-flow.md](../06-data-flow.md).

---

## 4. Kế hoạch tuần 3 (W3)

- Thiết kế sơ đồ mạng thử nghiệm (Lab Topology) chi tiết về địa chỉ IP, phân vùng mạng và giao diện bắt gói.
- Dựng cụm Kubernetes `k3s` và thiết lập các máy ảo/container đóng vai trò Attacker và Target.
