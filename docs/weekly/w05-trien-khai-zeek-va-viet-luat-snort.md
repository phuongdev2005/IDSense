# Báo cáo công việc Tuần 5 (W5): Triển khai Flow Extractor Zeek và xây dựng bộ luật Snort 3

- **Tuần thực hiện:** Tuần 5 (W5)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 5

1. Triển khai **Zeek** song song với Snort 3 để trích xuất thông tin luồng kết nối (`conn.log`) dưới định dạng JSON.
2. Xây dựng bộ luật chữ ký **Snort 3 (`local.rules`)** phát hiện 5 nhóm hành vi tấn công trong phạm vi đồ án:
   - ICMP Sweep (`ICMP_SWEEP`)
   - TCP Port Scan (`PORT_SCAN`)
   - SSH Brute Force (`BRUTE_FORCE`)
   - Web Attack (`WEB_ATTACK`: SQL Injection, XSS, Directory Traversal)
   - DoS / SYN Flood (`DOS`)
3. Kiểm tra tính hợp lệ và độ nhạy của bộ luật trong môi trường lab.

---

## 2. Nội dung thực hiện chi tiết

### 2.1. Cấu hình Zeek xuất `conn.log` định dạng JSON

Cấu hình tệp `local.zeek` để xuất toàn bộ log kết nối sang định dạng JSON với timestamp chuẩn ISO-8601:

```zeek
# /opt/zeek/share/zeek/site/local.zeek
@load policy/tuning/json-logs.zeek

redef LogAscii::use_json = T;
redef LogAscii::json_timestamps = JSON::TS_ISO8601;
```

Các trường chính trong `conn.log` của Zeek được sử dụng cho thuật toán thống kê (Lớp 1 & Lớp 2):

| Trường Zeek | Ý nghĩa | Vai trò trong thuật toán thống kê |
|---|---|---|
| `ts` | Thời điểm bắt đầu kết nối | Gom nhóm theo cửa sổ trượt 60 giây |
| `id.orig_h`, `id.orig_p` | IP nguồn và cổng nguồn | Định danh nguồn phát sinh lưu lượng |
| `id.resp_h`, `id.resp_p` | IP đích và cổng đích | Tính `unique_dst_ports` và `unique_dst_ips` |
| `proto`, `service` | Giao thức (tcp/udp/icmp) và dịch vụ (ssh, http, dns) | Phân loại ngữ cảnh kết nối |
| `duration` | Thời lượng kết nối (giây) | Phát hiện phiên kết nối ngắn bất thường |
| `orig_bytes`, `resp_bytes` | Số byte chiều đi và chiều về | Tính tốc độ byte và độ lệch Z-score |
| `orig_pkts`, `resp_pkts` | Số gói tin chiều đi và chiều về | Tính tốc độ gói tin phát hiện flood/DoS |
| `conn_state` | Trạng thái kết nối (`S0`, `SF`, `REJ`, `RSTO`) | Tính `failed_connection_ratio` phát hiện quét cổng |

### 2.2. Xây dựng bộ luật Snort 3 (`local.rules`)

Bộ luật tự xây dựng trong `/etc/snort/rules/local.rules`:

```snort
# 1. Phát hiện ICMP Sweep (gửi nhiều gói ICMP Echo Request trong 10 giây)
alert icmp any any -> $HOME_NET any (
    msg:"IDSENSE ICMP Echo Sweep Detected";
    itype:8;
    detection_filter:track by_src, count 10, seconds 10;
    classtype:attempted-recon;
    priority:3;
    sid:1000001; rev:1;
)

# 2. Phát hiện TCP SYN Port Scan
alert tcp any any -> $HOME_NET any (
    msg:"IDSENSE TCP SYN Port Scan Detected";
    flags:S;
    detection_filter:track by_src, count 20, seconds 10;
    classtype:attempted-recon;
    priority:2;
    sid:1000002; rev:1;
)

# 3. Phát hiện TCP NULL Scan
alert tcp any any -> $HOME_NET any (
    msg:"IDSENSE TCP NULL Scan Detected";
    flags:0;
    classtype:attempted-recon;
    priority:2;
    sid:1000003; rev:1;
)

# 4. Phát hiện TCP XMAS Scan (FIN, PSH, URG)
alert tcp any any -> $HOME_NET any (
    msg:"IDSENSE TCP XMAS Scan Detected";
    flags:FPU;
    classtype:attempted-recon;
    priority:2;
    sid:1000004; rev:1;
)

# 5. Phát hiện SSH Brute Force (nhiều kết nối TCP SYN mới tới cổng 22 trong 30 giây)
alert tcp any any -> $HOME_NET 22 (
    msg:"IDSENSE SSH Brute Force Attempt Detected";
    flags:S;
    detection_filter:track by_src, count 8, seconds 30;
    classtype:attempted-admin;
    priority:1;
    sid:1000005; rev:1;
)

# 6. Phát hiện Web SQL Injection cơ bản
alert tcp any any -> $HOME_NET $HTTP_PORTS (
    msg:"IDSENSE Web SQL Injection Attempt";
    flow:to_server,established;
    http_uri;
    content:"UNION",nocase;
    content:"SELECT",nocase;
    classtype:web-application-attack;
    priority:1;
    sid:1000006; rev:1;
)

# 7. Phát hiện Web Cross-Site Scripting (XSS)
alert tcp any any -> $HOME_NET $HTTP_PORTS (
    msg:"IDSENSE Web XSS Script Tag Attempt";
    flow:to_server,established;
    http_uri;
    content:"<script>",nocase;
    classtype:web-application-attack;
    priority:2;
    sid:1000007; rev:1;
)

# 8. Phát hiện Directory Traversal
alert tcp any any -> $HOME_NET $HTTP_PORTS (
    msg:"IDSENSE Web Directory Traversal Attempt";
    flow:to_server,established;
    http_uri;
    content:"../";
    classtype:web-application-attack;
    priority:2;
    sid:1000008; rev:1;
)

# 9. Phát hiện TCP SYN Flood / DoS
alert tcp any any -> $HOME_NET any (
    msg:"IDSENSE Possible TCP SYN Flood DoS";
    flags:S;
    detection_filter:track by_dst, count 200, seconds 5;
    classtype:attempted-dos;
    priority:1;
    sid:1000009; rev:1;
)
```

### 2.3. Kiểm tra cú pháp bộ luật

```bash
snort -c /etc/snort/snort.lua -R /etc/snort/rules/local.rules -T
```

Kết quả nạp thành công 9 rule không có lỗi cú pháp.

---

## 3. Sản phẩm đầu ra tuần 5

- Dịch vụ **Zeek** hoạt động trên cụm lab, xuất `conn.log` dạng JSON thời gian thực.
- Bộ luật **Snort 3 (`local.rules`)** gồm 9 rule bao phủ 5 nhóm kịch bản tấn công chính (`sid:1000001` – `sid:1000009`).

---

## 4. Kế hoạch tuần 6 (W6)

- Triển khai **Fluent Bit** để thu thập đồng thời `alert_json.txt` của Snort 3 và `conn.log` của Zeek.
- Chuẩn hóa tên trường dữ liệu JSON về một schema thống nhất.
- Chạy kiểm thử thực tế các kịch bản tấn công trong lab để xác nhận toàn bộ tầng Sensor + Collector hoạt động chính xác trước buổi báo cáo mốc W7.
