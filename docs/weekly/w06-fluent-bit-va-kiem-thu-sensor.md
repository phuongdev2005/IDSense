# Báo cáo công việc Tuần 6 (W6): Chuẩn hóa JSON Log bằng Fluent Bit và kiểm thử tầng Sensor

- **Tuần thực hiện:** Tuần 6 (W6)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 6

1. Thiết kế schema JSON chuẩn hóa cho sự kiện cảnh báo từ **Snort 3** và sự kiện luồng kết nối từ **Zeek**.
2. Cấu hình **Fluent Bit** đọc log thời gian thực (`tail`), chuẩn hóa tên trường (`modify`) và gắn nhãn nguồn sự kiện (`event_source`).
3. Thực hiện kiểm thử tích hợp tầng Sensor (Snort 3 + Zeek + Fluent Bit) bằng các công cụ tấn công trong mạng lab (`nmap`, `fping`, `hydra`, `curl`).

---

## 2. Nội dung thực hiện chi tiết

### 2.1. Cấu hình Fluent Bit (`fluent-bit.conf` và `parsers.conf`)

Fluent Bit thu nhận đồng thời 2 nguồn log JSON và ánh xạ các trường IP/Port về tên thống nhất (`src_ip`, `src_port`, `dst_ip`, `dst_port`):

```ini
# /etc/fluent-bit/fluent-bit.conf
[SERVICE]
    Flush        1
    Daemon       Off
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name         tail
    Path         /var/log/idsense/snort/alert_json.txt
    Tag          ids.snort
    Parser       json
    Refresh_Interval 1

[INPUT]
    Name         tail
    Path         /var/log/idsense/zeek/conn.log
    Tag          ids.zeek
    Parser       json
    Refresh_Interval 1

[FILTER]
    Name         modify
    Match        ids.snort
    Add          event_source snort
    Rename       src_addr src_ip
    Rename       dst_addr dst_ip

[FILTER]
    Name         modify
    Match        ids.zeek
    Add          event_source zeek
    Rename       id.orig_h src_ip
    Rename       id.orig_p src_port
    Rename       id.resp_h dst_ip
    Rename       id.resp_p dst_port

[OUTPUT]
    Name         stdout
    Match        ids.*
    Format       json_lines
```

### 2.2. Kiểm thử các kịch bản tấn công trên mạng Lab

Thực hiện phát sinh lưu lượng từ máy Attacker (`10.10.0.5`) tới máy Target (`10.10.0.20`):

| STT | Kịch bản kiểm thử | Lệnh thực thi trên Attacker (`10.10.0.5`) | Kết quả tại Snort 3 | Kết quả tại Zeek |
|---|---|---|---|---|
| 1 | **ICMP Sweep** | `fping -g 10.10.0.1 10.10.0.30 -r 0` | Sinh alert `sid:1000001` (`IDSENSE ICMP Echo Sweep Detected`) | Ghi nhận 30 bản ghi `proto: "icmp"` từ `10.10.0.5` |
| 2 | **TCP SYN Port Scan** | `nmap -sS -p 1-100 10.10.0.20` | Sinh alert `sid:1000002` (`IDSENSE TCP SYN Port Scan Detected`) | Ghi nhận 100 flow với `conn_state: "S0"` / `"REJ"` tới các `dst_port` khác nhau |
| 3 | **TCP XMAS Scan** | `nmap -sX -p 22,80,443 10.10.0.20` | Sinh alert `sid:1000004` (`IDSENSE TCP XMAS Scan Detected`) | Ghi nhận các flow TCP bất thường |
| 4 | **SSH Brute Force** | `hydra -l root -P pass.txt ssh://10.10.0.20 -t 4` | Sinh alert `sid:1000005` (`IDSENSE SSH Brute Force Attempt Detected`) | Ghi nhận nhiều phiên `service: "ssh"`, `dst_port: 22` liên tiếp |
| 5 | **Web SQLi & XSS** | `curl "http://10.10.0.20/vuln?id=1%20UNION%20SELECT%201,2"` | Sinh alert `sid:1000006` (`IDSENSE Web SQL Injection Attempt`) | Ghi nhận flow `service: "http"`, `dst_port: 80` |

### 2.3. Mẫu bản ghi JSON sau khi qua Fluent Bit

- **Bản ghi cảnh báo Snort 3 (`event_source: "snort"`):**
  ```json
  {
    "timestamp": "2026-10-15T14:20:11.402190Z",
    "event_source": "snort",
    "proto": "TCP",
    "src_ip": "10.10.0.5",
    "src_port": 49812,
    "dst_ip": "10.10.0.20",
    "dst_port": 22,
    "sid": 1000002,
    "msg": "IDSENSE TCP SYN Port Scan Detected",
    "priority": 2,
    "class": "Attempted Information Leak",
    "action": "allow"
  }
  ```

- **Bản ghi flow Zeek (`event_source: "zeek"`):**
  ```json
  {
    "ts": "2026-10-15T14:20:11.401850Z",
    "event_source": "zeek",
    "uid": "C8xKjM2nPqR9sT1vW3",
    "src_ip": "10.10.0.5",
    "src_port": 49812,
    "dst_ip": "10.10.0.20",
    "dst_port": 22,
    "proto": "tcp",
    "service": "-",
    "duration": 0.0012,
    "orig_bytes": 0,
    "resp_bytes": 0,
    "conn_state": "S0",
    "orig_pkts": 1,
    "resp_pkts": 0
  }
  ```

---

## 3. Sản phẩm đầu ra tuần 6

- Cấu hình **Fluent Bit** hoàn chỉnh, chuẩn hóa schema JSON cho cả Snort 3 và Zeek.
- Kết quả kiểm thử thực tế 5 kịch bản tấn công trong mạng lab với đầy đủ log mẫu.

---

## 4. Kế hoạch tuần 7 (W7)

- Rà soát toàn bộ tài liệu và môi trường demo Giai đoạn 1 (Tuần 1 – Tuần 7).
- Thực hiện **Báo cáo tiến độ Đợt 1 vào Thứ Năm, W7, khung giờ 20:30–22:00**.
