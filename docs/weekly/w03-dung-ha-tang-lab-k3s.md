# Báo cáo công việc Tuần 3 (W3): Thiết kế sơ đồ mạng Lab và Hướng dẫn dựng hạ tầng Kubernetes (k3s)

- **Tuần thực hiện:** Tuần 3 (W3)
- **Giai đoạn:** Đợt 1 (W1 – W7, hướng tới mốc báo cáo **Thu, W7, 20:30–22:00**)
- **Trạng thái:** Hoàn thành

---

## 1. Mục tiêu tuần 3

1. Thiết kế sơ đồ mạng lab cô lập, an toàn trên dải mạng nội bộ `10.10.0.0/24`.
2. Cài đặt và cấu hình cụm Kubernetes nhẹ (**k3s**) cùng công cụ quản trị `kubectl`, `helm`.
3. Dựng mạng Linux Bridge `br-idsense` (`10.10.0.1/24`) đóng vai trò switch/hub ảo để cảm biến thu nhận toàn bộ lưu lượng giữa máy tấn công (**Attacker – `10.10.0.5`**) và máy mục tiêu (**Target – `10.10.0.20`**).
4. Kiểm chứng khả năng bắt gói tin thực tế trên giao diện `br-idsense` từ bên trong Pod của Kubernetes.

---

## 2. Sơ đồ kiến trúc mạng Lab trên máy Ubuntu

Mô hình dưới đây cho phép dựng toàn bộ môi trường Lab ngay trên **1 máy Ubuntu (Ubuntu 22.04 / 24.04)** vừa tiết kiệm RAM, vừa đảm bảo Snort 3 và Zeek chạy trên `k3s` bắt được 100% gói tin đi qua mạng lab:

```text
+-------------------------------------------------------------------------------+
|                     Máy Host Ubuntu (k3s Node - 10.10.0.1)                    |
|                                                                               |
|  +-------------------------+                     +-------------------------+  |
|  |   Attacker Container    |   Gói tin tấn công  |    Target Container     |  |
|  |       10.10.0.5         | ------------------> |       10.10.0.20        |  |
|  | - nmap, fping, hydra    |                     | - DVWA Web App (:80)    |  |
|  | - curl, tcpreplay       | <------------------ | - OpenSSH Server (:22)  |  |
|  +------------+------------+                     +------------+------------+  |
|               |                                               |               |
|               +-------------------+       +-------------------+               |
|                                   |       |                                   |
|                                   v       v                                   |
|                     +-----------------------------------+                     |
|                     |   Linux Bridge: br-idsense        |                     |
|                     |   Subnet: 10.10.0.0/24            |                     |
|                     |   Mode: PROMISC + ageing_time=0   |                     |
|                     +-----------------+-----------------+                     |
|                                       |                                       |
|                                       | AF_PACKET / Raw Socket (-i br-idsense)|
|                                       v                                       |
|  +-------------------------------------------------------------------------+  |
|  |                       Cụm Kubernetes (k3s Cluster)                      |  |
|  |  Namespace: idsense-sensor                                              |  |
|  |   - Pod Snort 3 Sensor (hostNetwork: true, interface: br-idsense)       |  |
|  |   - Pod Zeek Flow Extractor (hostNetwork: true, interface: br-idsense)  |  |
|  |   - Pod Fluent Bit Log Collector                                        |  |
|  |  Namespace: idsense-pipeline   (Kafka, Anomaly Detection, Correlation)  |  |
|  |  Namespace: idsense-monitoring (Elasticsearch, Grafana, Alertmanager)   |  |
|  +-------------------------------------------------------------------------+  |
+-------------------------------------------------------------------------------+
```

### Quy hoạch địa chỉ IP trong Lab

| Thành phần | Địa chỉ IP | Giao diện / Runtime | Vai trò & Công cụ |
|---|---|---|---|
| **Gateway / Host Bridge** | `10.10.0.1/24` | `br-idsense` (Linux Bridge) | Cầu nối mạng lab; cổng lắng nghe (`-i br-idsense`) của Snort 3 và Zeek |
| **Attacker Node** | `10.10.0.5` | Container `idsense-attacker` | Phát sinh lưu lượng bình thường & tấn công (`nmap`, `fping`, `hydra`, `curl`, `tcpreplay`) |
| **Target Node** | `10.10.0.20` | Container `idsense-target` | Máy chủ đích chạy Web DVWA (`:80`) và OpenSSH (`:22`) |
| **k3s Cluster** | `10.10.0.1` (host) | Dịch vụ `k3s.service` | Quản lý các Pod Sensor, Pipeline và Monitoring |

---

## 3. Hướng dẫn từng bước dựng Lab k3s (Step-by-step)

### Bước 1: Cài đặt các gói công cụ phụ trợ trên Ubuntu

```bash
sudo apt-get update
sudo apt-get install -y curl wget git jq ethtool net-tools iproute2 tcpdump bridge-utils
```

Cài đặt **Helm** (trình quản lý gói cho Kubernetes):

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

### Bước 2: Tạo mạng Bridge cô lập `br-idsense` (`10.10.0.0/24`)

Sử dụng Docker để tạo một mạng bridge có tên giao diện cố định trên hệ điều hành là `br-idsense`:

```bash
docker network create \
  --driver=bridge \
  --subnet=10.10.0.0/24 \
  --gateway=10.10.0.1 \
  -o com.docker.network.bridge.name=br-idsense \
  idsense-lab-net
```

Cấu hình giao diện `br-idsense` ở chế độ **Promiscuous**, đặt `ageing_time 0` (để bridge chuyển tiếp bản sao gói tin unicast tới socket bắt gói của Snort/Zeek giống như cổng SPAN/Hub) và tắt tính năng gộp gói phần cứng (GRO/LRO/TSO/GSO):

```bash
# 1. Bật chế độ Promiscuous
sudo ip link set dev br-idsense promisc on

# 2. Đặt ageing_time = 0 để mọi gói tin giữa 10.10.0.5 và 10.10.0.20 đều hiển thị cho sensor
sudo ip link set dev br-idsense type bridge ageing_time 0

# 3. Tắt packet offloading để Snort 3 và Zeek đọc đúng kích thước gói tin
sudo ethtool -K br-idsense gro off lro off tso off gso off 2>/dev/null || true

# 4. Kiểm tra trạng thái giao diện br-idsense
ip -d link show dev br-idsense
```

> **Kiểm tra:** Trong kết quả trả về của lệnh `ip -d link show dev br-idsense` phải có cờ **`PROMISC`**, địa chỉ `10.10.0.1/24` và `ageing_time 0`.

---

### Bước 3: Cài đặt cụm Kubernetes `k3s`

1. **Chạy script cài đặt `k3s` (cấp quyền đọc file kubeconfig cho user hiện tại):**
   ```bash
   curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
   ```

2. **Cấu hình biến môi trường `~/.kube/config` để dùng `kubectl` và `helm` không cần `sudo`:**
   ```bash
   mkdir -p ~/.kube
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   sudo chown $(id -u):$(id -g) ~/.kube/config
   chmod 600 ~/.kube/config
   export KUBECONFIG=~/.kube/config
   ```
   *(Nên thêm dòng `export KUBECONFIG=~/.kube/config` vào cuối file `~/.bashrc`)*.

3. **Kiểm tra trạng thái Node và các Pod hệ thống của `k3s`:**
   ```bash
   kubectl get nodes -o wide
   kubectl get pods -A
   ```
   Đợi khoảng 20–30 giây cho đến khi node ở trạng thái `Ready`.

4. **Khởi tạo 3 Namespace cho dự án IDSense:**
   ```bash
   kubectl create namespace idsense-sensor
   kubectl create namespace idsense-pipeline
   kubectl create namespace idsense-monitoring
   kubectl get namespaces | grep idsense
   ```

5. **Tạo thư mục lưu trữ log dùng chung trên Host cho Snort 3, Zeek và Fluent Bit:**
   ```bash
   sudo mkdir -p /var/log/idsense/snort /var/log/idsense/zeek
   sudo chmod -R 777 /var/log/idsense
   ```

---

### Bước 4: Khởi tạo máy mục tiêu (`10.10.0.20`) và máy tấn công (`10.10.0.5`)

#### 4.1. Khởi chạy máy mục tiêu `idsense-target` (`10.10.0.20`)

Chạy ứng dụng web thử nghiệm **DVWA** tại địa chỉ `10.10.0.20` và cài thêm dịch vụ `openssh-server` để phục vụ kịch bản kiểm thử SSH Brute Force:

```bash
docker run -d \
  --name idsense-target \
  --hostname target-node \
  --network idsense-lab-net \
  --ip 10.10.0.20 \
  --restart unless-stopped \
  vulnerables/web-dvwa

# Cài đặt và bật thêm dịch vụ SSH (cổng 22) bên trong máy mục tiêu
docker exec -u 0 idsense-target bash -c "
  apt-get update &&
  apt-get install -y openssh-server &&
  mkdir -p /var/run/sshd &&
  echo 'root:labpassword123' | chpasswd &&
  sed -i 's/#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config &&
  service ssh start
"
```

#### 4.2. Khởi chạy máy tấn công `idsense-attacker` (`10.10.0.5`)

Chạy container Ubuntu tại địa chỉ `10.10.0.5` và cài đặt bộ công cụ kiểm thử (`nmap`, `fping`, `hydra`, `curl`, `tcpreplay`, `iputils-ping`):

```bash
docker run -d \
  --name idsense-attacker \
  --hostname attacker-node \
  --network idsense-lab-net \
  --ip 10.10.0.5 \
  --cap-add=NET_RAW \
  --cap-add=NET_ADMIN \
  --restart unless-stopped \
  ubuntu:24.04 sleep infinity

# Cài đặt các công cụ tạo lưu lượng và kiểm thử xâm nhập
docker exec idsense-attacker bash -c "
  apt-get update &&
  DEBIAN_FRONTEND=noninteractive apt-get install -y \
    iputils-ping iproute2 curl wget nmap fping hydra tcpreplay netcat-openbsd dnsutils
"
```

---

### Bước 5: Kiểm tra kết nối và bắt gói tin từ Pod trên `k3s`

#### 5.1. Kiểm tra kết nối từ Attacker (`10.10.0.5`) sang Target (`10.10.0.20`)

```bash
# Kiểm tra ping ICMP
docker exec idsense-attacker ping -c 3 10.10.0.20

# Kiểm tra cổng 22 (SSH) và cổng 80 (HTTP DVWA)
docker exec idsense-attacker nmap -sS -p 22,80 10.10.0.20
```

#### 5.2. Kiểm tra bắt gói tin từ một Pod Sensor thử nghiệm trên `k3s`

Chạy một Pod tạm thời trong namespace `idsense-sensor` với `hostNetwork: true` để xác nhận các Pod trên `k3s` nhìn thấy toàn bộ gói tin trên giao diện `br-idsense`:

```bash
kubectl -n idsense-sensor run packet-test \
  --rm -i --tty \
  --overrides='{"spec":{"hostNetwork":true}}' \
  --image=nicolaka/netshoot \
  -- tcpdump -i br-idsense -nn net 10.10.0.0/24 -c 5
```

Trong lúc lệnh trên đang chờ, mở một terminal khác và chạy:

```bash
docker exec idsense-attacker ping -c 5 10.10.0.20
```

Pod `packet-test` trên `k3s` sẽ bắt được đủ 5 gói tin `10.10.0.5 > 10.10.0.20: ICMP echo request` và tự động thoát. Điều này chứng minh hạ tầng mạng Lab + `k3s` đã sẵn sàng 100% cho **Snort 3** và **Zeek**!

---

## 4. Các lệnh quản trị nhanh môi trường Lab

| Thao tác | Lệnh thực hiện |
|---|---|
| Truy cập shell máy Attacker (`10.10.0.5`) | `docker exec -it idsense-attacker bash` |
| Truy cập shell máy Target (`10.10.0.20`) | `docker exec -it idsense-target bash` |
| Xem danh sách Pod trong cụm `k3s` | `kubectl get pods -A -o wide` |
| Tạm dừng dịch vụ `k3s` khi không dùng | `sudo /usr/local/bin/k3s-killall.sh` |
| Khởi động lại dịch vụ `k3s` | `sudo systemctl start k3s` |
| Bật lại cờ `PROMISC` sau khi khởi động lại máy | `sudo ip link set dev br-idsense promisc on && sudo ip link set dev br-idsense type bridge ageing_time 0` |

---

## 5. Sản phẩm đầu ra tuần 3

- Mạng lab `br-idsense` (`10.10.0.0/24`) hoạt động ở chế độ `PROMISC` với `ageing_time 0`.
- Cụm Kubernetes `k3s` hoạt động ổn định với 3 namespace (`idsense-sensor`, `idsense-pipeline`, `idsense-monitoring`).
- Hai node thử nghiệm `idsense-attacker` (`10.10.0.5`) và `idsense-target` (`10.10.0.20` – chạy SSH `:22` và DVWA `:80`) sẵn sàng phát sinh kịch bản lưu lượng.

---

## 6. Kế hoạch tuần 4 (W4)

- Đóng gói container image cho **Snort 3**.
- Cấu hình `snort.lua` lắng nghe trên giao diện `br-idsense` và xuất cảnh báo định dạng `alert_json`.
- Triển khai `DaemonSet` Snort 3 lên namespace `idsense-sensor` của cụm `k3s`.
