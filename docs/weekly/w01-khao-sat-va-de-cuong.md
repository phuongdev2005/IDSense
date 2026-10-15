# Báo cáo công việc Tuần 1 (W1): Khảo sát bài toán, nghiên cứu Snort 3 và chốt đề cương

- **Tuần thực hiện:** Tuần 1 (W1)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 1

1. Khảo sát tổng quan về Hệ thống phát hiện xâm nhập mạng (Network Intrusion Detection System – NIDS).
2. Nghiên cứu nguyên lý hoạt động và kiến trúc xử lý gói tin của **Snort 3**.
3. Phân tích ưu, nhược điểm của phương pháp phát hiện dựa trên chữ ký (Signature-based) và phương pháp phát hiện bất thường dựa trên thống kê (Statistical Anomaly-based).
4. Xác định rõ tên đề tài, mục tiêu, phạm vi và đặc tả yêu cầu hệ thống (không sử dụng AI/ML).

---

## 2. Nội dung thực hiện chi tiết

### 2.1. Khảo sát phân loại hệ thống IDS

| Tiêu chí | Phát hiện theo chữ ký (Signature-based – Snort 3) | Phát hiện bất thường thống kê (Statistical Anomaly-based) |
|---|---|---|
| **Nguyên lý** | Đối chiếu header và payload gói tin với tập luật (rules/signatures) đã biết | Đo lường hành vi lưu lượng (flow) theo cửa sổ thời gian và so sánh với ngưỡng hoặc đường cơ sở (baseline) |
| **Ưu điểm** | Độ chính xác cao với các mẫu tấn công đã biết; tỷ lệ cảnh báo giả (FPR) thấp; giải thích rõ rule khớp | Phát hiện được hành vi quét, vét cạn, flood hoặc lưu lượng bất thường ngay cả khi payload bị mã hóa hoặc chưa có rule chữ ký cụ thể |
| **Hạn chế** | Bỏ sót nếu biến thể tấn công thay đổi mẫu payload hoặc chưa kịp cập nhật rule | Dễ sinh cảnh báo giả nếu lưu lượng hợp lệ tăng đột biến và ngưỡng cấu hình chưa phù hợp |
| **Vai trò trong đồ án** | Lớp phòng tuyến thứ nhất bắt các mẫu tấn công tường minh ở mức gói tin | Lớp phân tích bổ trợ ở mức luồng kết nối (flow) bằng luật ngưỡng trượt và độ lệch chuẩn Z-score |

### 2.2. Nghiên cứu kiến trúc Snort 3

So với Snort 2, **Snort 3** được thiết kế lại với các cải tiến quan trọng phục vụ triển khai hiện đại:

1. **Hỗ trợ đa luồng (Multi-threading):** nhiều packet processing thread xử lý đồng thời trên một tiến trình duy nhất, giảm hiện tượng rớt gói khi lưu lượng tăng.
2. **Cấu hình bằng Lua (`snort.lua`):** cú pháp rõ ràng, dễ quản lý dưới dạng mã nguồn (Configuration as Code) và đóng gói vào `ConfigMap` trên Kubernetes.
3. **Luồng xử lý gói tin trong Snort 3:**
   ```text
   NIC / Traffic Mirror
        ↓
   DAQ (Data Acquisition Library: afpacket / pcap)
        ↓
   Packet Decoder (Ethernet → IPv4 → TCP/UDP/ICMP)
        ↓
   Stream & Service Inspectors (reassembly TCP stream, parse HTTP/SSH/DNS)
        ↓
   Detection Engine (đối chiếu bộ luật local.rules / community rules)
        ↓
   Output Logger (xuất cảnh báo định dạng alert_json)
   ```

### 2.3. Xác định phạm vi và yêu cầu hệ thống

- **Phạm vi mạng giám sát:** Mạng IPv4 trong môi trường lab được kiểm soát và cấp phép; hệ thống hoạt động ở chế độ **IDS (chỉ giám sát và cảnh báo)**, không tự động chặn gói tin.
- **Các hành vi tấn công trọng tâm:**
  1. Quét cổng mạng (`PORT_SCAN` – TCP SYN/FIN/XMAS scan).
  2. Dò quét host bằng ICMP (`ICMP_SWEEP`).
  3. Tấn công vét cạn mật khẩu SSH (`BRUTE_FORCE`).
  4. Tấn công ứng dụng web cơ bản (`WEB_ATTACK` – SQL Injection, XSS, Path Traversal).
  5. Tấn công từ chối dịch vụ mô phỏng giới hạn (`DOS` / Flood).
- **Định hướng thuật toán:** Sử dụng **Snort 3** kết hợp **thuật toán thống kê và luật ngưỡng (Sliding-window Threshold, Z-score, Event Correlation)** trên dữ liệu flow từ Zeek; **không sử dụng mô hình AI/ML**.

---

## 3. Sản phẩm đầu ra tuần 1

- Hoàn thiện tài liệu [01-de-cuong.md](../01-de-cuong.md): xác định tên đề tài, bối cảnh, mục tiêu, phạm vi và kết quả dự kiến.
- Hoàn thiện tài liệu [02-dac-ta-yeu-cau.md](../02-dac-ta-yeu-cau.md): xác định 13 yêu cầu chức năng (`FR-01` đến `FR-13`), các yêu cầu phi chức năng và cấu trúc dữ liệu sự kiện tối thiểu.

---

## 4. Kế hoạch tuần 2 (W2)

- Thiết kế kiến trúc tổng thể 6 lớp của hệ thống IDSense.
- Chốt bảng công nghệ cố định (Tech Stack) và công thức tính điểm rủi ro (`risk_score`).
- Thiết kế luồng dữ liệu (Data Flow) và định dạng bản ghi JSON xuyên suốt hệ thống.
