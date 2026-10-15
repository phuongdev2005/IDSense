# Báo cáo Tiến độ Đợt 1 (Tuần 7 – W7)

- **Mốc báo cáo:** Báo cáo tiến độ Đợt 1 (**Milestone W7**)
- **Thời gian báo cáo:** **Thứ Năm (Thu), Tuần 7 (W7), 20:30 – 22:00**
- **Giai đoạn tổng hợp:** Tuần 1 – Tuần 7 (W1 – W7)
- **Đề tài:** Xây dựng hệ thống giám sát an ninh mạng sử dụng Snort và các thuật toán phát hiện xâm nhập

---

## 1. Tóm tắt tiến độ Giai đoạn 1 (W1 – W7)

| Tuần | Nội dung công việc theo kế hoạch | Kết quả đạt được | Trạng thái |
|---|---|---|---|
| **W1** | Khảo sát bài toán NIDS, nghiên cứu Snort 3 và chốt đề cương, đặc tả yêu cầu | Hoàn thành [01-de-cuong.md](../01-de-cuong.md), [02-dac-ta-yeu-cau.md](../02-dac-ta-yeu-cau.md), [Báo cáo W1](w01-khao-sat-va-de-cuong.md) | Hoàn thành 100% |
| **W2** | Thiết kế kiến trúc 6 lớp, luồng dữ liệu và chốt quyết định kỹ thuật (không dùng AI/ML) | Hoàn thành [03-thiet-ke-kien-truc.md](../03-thiet-ke-kien-truc.md), [04-ke-hoach-thuc-hien.md](../04-ke-hoach-thuc-hien.md), [05-quyet-dinh-ky-thuat.md](../05-quyet-dinh-ky-thuat.md), [06-data-flow.md](../06-data-flow.md), [Báo cáo W2](w02-kien-truc-va-quyet-dinh-ky-thuat.md) | Hoàn thành 100% |
| **W3** | Thiết kế sơ đồ mạng lab `10.10.0.0/24` và dựng cụm Kubernetes `k3s` | Cụm `k3s` hoạt động, cấu hình card mạng `PROMISC` và máy Attacker/Target ([Báo cáo W3](w03-dung-ha-tang-lab-k3s.md)) | Hoàn thành 100% |
| **W4** | Đóng gói và triển khai cảm biến Snort 3 trên Kubernetes | `DaemonSet` Snort 3 chạy trên `k3s`, xuất cảnh báo `alert_json` ([Báo cáo W4](w04-trien-khai-snort3-tren-k8s.md)) | Hoàn thành 100% |
| **W5** | Triển khai Flow Extractor Zeek và viết bộ luật chữ ký Snort 3 (`local.rules`) | Zeek xuất `conn.log` JSON; hoàn thiện 9 rule Snort 3 cho 5 nhóm tấn công ([Báo cáo W5](w05-trien-khai-zeek-va-viet-luat-snort.md)) | Hoàn thành 100% |
| **W6** | Chuẩn hóa log JSON bằng Fluent Bit và kiểm thử tầng Sensor | Fluent Bit gom và chuẩn hóa schema JSON từ Snort 3 + Zeek ([Báo cáo W6](w06-fluent-bit-va-kiem-thu-sensor.md)) | Hoàn thành 100% |
| **W7** | Tổng hợp kết quả Giai đoạn 1 và báo cáo tiến độ Đợt 1 (**Thu, W7, 20:30–22:00**) | Hoàn thiện tài liệu tổng hợp và kịch bản demo trực tiếp | Hoàn thành 100% |

---

## 2. Kết quả trọng tâm đạt được trong Đợt 1

### 2.1. Chốt kiến trúc và phương pháp phát hiện (Không dùng AI/ML)

- **Cảm biến kép (Dual Sensor):**
  - **Snort 3:** phát hiện các mẫu tấn công đã biết dựa trên chữ ký gói tin (`alert_json`).
  - **Zeek:** trích xuất thông tin luồng kết nối (`conn.log`) phục vụ phát hiện bất thường ngay cả khi payload mã hóa hoặc chưa có rule tĩnh.
- **Ba lớp thuật toán phát hiện xâm nhập (sẽ cài đặt ở Đợt 2 & Đợt 3):**
  1. **Lớp 1 – Luật ngưỡng tần suất trên cửa sổ trượt 60 giây (`threshold_score`):** phát hiện `PORT_SCAN`, `ICMP_SWEEP`, `BRUTE_FORCE`, `DOS`.
  2. **Lớp 2 – Độ lệch chuẩn thống kê Z-score (`anomaly_score`):** đo độ lệch $z = \frac{x - \mu_{\text{normal}}}{\sigma_{\text{normal}}}$ so với đường cơ sở lưu lượng bình thường.
  3. **Lớp 3 – Hợp nhất tương quan (`Correlation Engine`):**
     $$\text{risk\_score} = 0.45 \times \text{snort\_score} + 0.35 \times \text{threshold\_score} + 0.20 \times \text{anomaly\_score}$$

### 2.2. Hạ tầng Lab và Tầng Sensor đã triển khai

- **Mạng Lab (`10.10.0.0/24`):**
  - `10.10.0.5`: Máy Attacker / Traffic Generator (`nmap`, `fping`, `hydra`, `curl`, `tcpreplay`).
  - `10.10.0.20`: Máy Target (OpenSSH, ứng dụng web thử nghiệm DVWA).
  - `10.10.0.10`: Cụm Kubernetes `k3s` chạy `snort3-sensor`, `zeek` và `fluent-bit`.
- **Bộ luật Snort 3 (`local.rules`):** Đã viết và kiểm thử 9 rule (`sid:1000001` đến `sid:1000009`) phát hiện ICMP Sweep, TCP SYN/NULL/XMAS Port Scan, SSH Brute Force, SQL Injection, XSS, Directory Traversal và SYN Flood.

---

## 3. Kịch bản Demo trực tiếp tại buổi báo cáo (Thu, W7, 20:30–22:00)

### Bước 1: Kiểm tra hạ tầng Kubernetes `k3s` và trạng thái các Pod Sensor

```bash
kubectl get nodes -o wide
kubectl -n idsense-sensor get pods -o wide
```

### Bước 2: Mở luồng quan sát Log JSON chuẩn hóa từ Fluent Bit

```bash
kubectl -n idsense-sensor logs -f daemonset/fluent-bit
```

### Bước 3: Thực hiện 3 kịch bản tấn công mẫu từ máy Attacker (`10.10.0.5`)

1. **Dò quét mạng bằng ICMP Sweep:**
   ```bash
   fping -g 10.10.0.1 10.10.0.30 -r 0
   ```
   → Quan sát cảnh báo `sid:1000001` từ Snort 3 và loạt bản ghi ICMP flow từ Zeek.

2. **Quét cổng TCP SYN bằng Nmap:**
   ```bash
   nmap -sS -p 1-100 10.10.0.20
   ```
   → Quan sát cảnh báo `sid:1000002` từ Snort 3 và các flow `conn_state: "S0"`/`"REJ"` từ Zeek.

3. **Tấn công thử nghiệm SQL Injection vào Web Server:**
   ```bash
   curl -i "http://10.10.0.20/vulnerabilities/sqli/?id=1%27%20UNION%20SELECT%201,2--+-&Submit=Submit"
   ```
   → Quan sát cảnh báo `sid:1000006` (`IDSENSE Web SQL Injection Attempt`) kèm thông tin HTTP flow.

---

## 4. Vấn đề gặp phải và cách giải quyết trong Đợt 1

| Vấn đề | Nguyên nhân | Giải pháp đã áp dụng |
|---|---|---|
| Snort 3 không nhìn thấy gói tin đi giữa hai máy khác trong mạng lab | Giao diện mạng bridge trên node k3s chưa bật chế độ Promiscuous, `ageing_time` mặc định > 0 và bị bật tính năng GRO/LRO | Bật `ip link set dev br-idsense promisc on`, đặt `ageing_time 0` và tắt `gro/lro/tso/gso` bằng `ethtool` |
| Trường địa chỉ IP giữa Snort 3 (`src_addr`, `dst_addr`) và Zeek (`id.orig_h`, `id.resp_h`) khác tên | Mỗi công cụ sử dụng quy ước đặt tên trường JSON riêng | Sử dụng filter `modify` trong Fluent Bit để đổi tên thống nhất về `src_ip`, `src_port`, `dst_ip`, `dst_port` |

---

## 5. Kế hoạch Giai đoạn 2 (Tuần 8 – Tuần 10)

Hướng tới **Báo cáo tiến độ Đợt 2 vào Thứ Năm, Tuần 10 (Thu, W10, 20:30–22:00)**:

- **Tuần 8 (W8):** Triển khai **Kafka** làm event bus, kết nối **Elasticsearch** và dựng dashboard **Grafana** cơ bản để lưu trữ và hiển thị cảnh báo/flow.
- **Tuần 9 (W9):** Thu thập lưu lượng bình thường trong mạng lab để xây dựng đường cơ sở (**Baseline**: tính $\mu_{\text{normal}}, \sigma_{\text{normal}}$); cài đặt bộ gom đặc trưng theo cửa sổ trượt 60 giây.
- **Tuần 10 (W10):** Cài đặt và kiểm thử thuật toán **Luật ngưỡng tần suất (Lớp 1)** và **Phát hiện bất thường thống kê Z-score (Lớp 2)**; chuẩn bị báo cáo và demo mốc **W10 (Thu, 20:30–22:00)**.
