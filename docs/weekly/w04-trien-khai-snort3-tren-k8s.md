# Báo cáo công việc Tuần 4 (W4): Đóng gói và triển khai cảm biến Snort 3 trên Kubernetes

- **Tuần thực hiện:** Tuần 4 (W4)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 4

1. Xây dựng cấu hình chuẩn cho **Snort 3** (`snort.lua`) phù hợp với mạng lab `10.10.0.0/24`.
2. Cấu hình module xuất cảnh báo dạng JSON (`alert_json`) với đầy đủ các trường cần thiết cho bước chuẩn hóa và tương quan.
3. Triển khai Snort 3 dưới dạng `DaemonSet` trên Kubernetes (`k3s`) sử dụng `hostNetwork: true` và quyền `CAP_NET_RAW`, `CAP_NET_ADMIN`.
4. Kiểm thử khả năng bắt gói tin trực tiếp từ giao diện mạng của pod Snort 3.

---

## 2. Nội dung thực hiện chi tiết

### 2.1. Cấu hình `snort.lua` và module `alert_json`

Thiết lập tệp cấu hình `/etc/snort/snort.lua` với dải mạng giám sát `HOME_NET` và đầu ra JSON chuẩn hóa:

```lua
-- /etc/snort/snort.lua
HOME_NET = '10.10.0.0/24'
EXTERNAL_NET = 'any'

include 'snort_defaults.lua'

stream = { }
stream_ip = { }
stream_icmp = { }
stream_tcp = { }
stream_udp = { }
http_inspect = { }

ips = {
    enable_builtin_rules = false,
    include = '/etc/snort/rules/local.rules',
}

alert_json = {
    file = true,
    limit = 100,
    fields = 'timestamp pkt_num proto pkt_gen pkt_len dir src_ap dst_ap src_addr src_port dst_addr dst_port rule action msg priority class sid rev',
}
```

### 2.2. Thiết kế Manifest Kubernetes cho Snort 3 (`DaemonSet`)

Snort 3 được triển khai dưới dạng `DaemonSet` trong namespace `idsense-sensor`, chia sẻ thư mục `/var/log/snort` với Fluent Bit sidecar hoặc qua `hostPath`/`emptyDir`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: snort3-sensor
  namespace: idsense-sensor
  labels:
    app: snort3-sensor
spec:
  selector:
    matchLabels:
      app: snort3-sensor
  template:
    metadata:
      labels:
        app: snort3-sensor
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: snort3
          image: idsense/snort3:3.1
          command:
            - snort
            - -c
            - /etc/snort/snort.lua
            - -i
            - br-idsense
            - -l
            - /var/log/snort
          securityContext:
            capabilities:
              add:
                - NET_ADMIN
                - NET_RAW
          volumeMounts:
            - name: snort-config
              mountPath: /etc/snort
            - name: snort-logs
              mountPath: /var/log/snort
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
      volumes:
        - name: snort-config
          configMap:
            name: snort3-config
        - name: snort-logs
          hostPath:
            path: /var/log/idsense/snort
            type: DirectoryOrCreate
```

### 2.3. Kiểm thử vận hành Snort 3 trên Kubernetes

1. **Kiểm tra cấu hình hợp lệ trước khi chạy:**
   ```bash
   snort -c /etc/snort/snort.lua -T
   ```
   Kết quả báo `Snort successfully validated the configuration (with 0 warnings)`.

2. **Kiểm tra trạng thái Pod và file log sinh ra:**
   ```bash
   kubectl -n idsense-sensor get pods -l app=snort3-sensor
   kubectl -n idsense-sensor logs daemonset/snort3-sensor
   ls -lh /var/log/idsense/snort/alert_json.txt
   ```

---

## 3. Sản phẩm đầu ra tuần 4

- Image container và cấu hình `snort.lua` xuất `alert_json.txt`.
- Manifest Kubernetes `DaemonSet` và `ConfigMap` cho cảm biến Snort 3 chạy ổn định trên cụm `k3s`.
- Xác nhận Snort 3 bắt được lưu lượng trên giao diện `br-idsense` và ghi cảnh báo JSON ra đĩa.

---

## 4. Kế hoạch tuần 5 (W5)

- Triển khai **Zeek** để trích xuất siêu dữ liệu kết nối (`conn.log`) dưới dạng JSON.
- Xây dựng bộ luật chữ ký Snort 3 (`local.rules`) phát hiện các kịch bản: Port Scan, ICMP Sweep, SSH Brute Force, Web Attack (SQLi/XSS/Traversal) và DoS.
