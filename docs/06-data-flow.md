# 6. Data flow của hệ thống

## 6.1. Một traffic đi qua hệ thống như thế nào?

```text
1. Máy nguồn gửi packet
        ↓
2. Snort 3 đọc packet và kiểm tra rule
        ↓
3. Zeek gom packet thành connection/flow
        ↓
4. Fluent Bit chuẩn hóa log JSON
        ↓
5. Kafka nhận event
        ↓
6. ML service chạy luật ngưỡng, Z-score và Decision Tree để tạo ml_label và anomaly_score
        ↓
7. Correlation tạo incident và risk_score
        ↓
8. Elasticsearch lưu kết quả
        ↓
9. Grafana hiển thị dashboard
        ↓
10. Alertmanager gửi thông báo nếu vượt ngưỡng
```

## 6.2. Ví dụ port scan

### Flow từ Zeek

```json
{
  "ts": "2026-10-08T10:01:02Z",
  "src_ip": "10.10.0.5",
  "dst_ip": "10.10.0.20",
  "service": "-",
  "duration": 0.02,
  "orig_bytes": 0,
  "resp_bytes": 0,
  "conn_state": "S0",
  "dst_port": 22,
  "proto": "tcp"
}
```

Trong cửa sổ 60 giây, hệ thống tổng hợp thêm:

```json
{
  "unique_dst_ports": 42,
  "connection_count": 42,
  "syn_count": 42,
  "failed_connection_ratio": 0.98
}
```

### Kết quả Snort

```json
{
  "signature": "TCP port scan",
  "severity": 2,
  "src_ip": "10.10.0.5",
  "dst_ip": "10.10.0.20"
}
```

### Kết quả ML

```json
{
  "ml_label": "PORT_SCAN",
  "ml_probability": 0.96,
  "anomaly_score": 0.91,
  "top_features": ["unique_dst_ports", "syn_count", "failed_connection_ratio"]
}
```

### Incident cuối cùng

```json
{
  "incident_type": "PORT_SCAN",
  "risk_score": 94.2,
  "severity": "high",
  "evidence": ["snort_signature", "high_ml_probability", "abnormal_connection_rate"]
}
```

## 6.3. Ý nghĩa của từng loại dữ liệu

| Loại | Ví dụ | Dùng để |
|---|---|---|
| Packet/PCAP | TCP SYN, DNS query | Replay và điều tra sâu |
| Flow | duration, bytes, packets | Đầu vào cho ML |
| Snort alert | signature, severity | Phát hiện theo rule |
| ML result | label, probability | Phát hiện bất thường/phân loại |
| Incident | risk score, evidence | Hiển thị và cảnh báo |
