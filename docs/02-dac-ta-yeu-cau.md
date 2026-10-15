# 2. Đặc tả yêu cầu hệ thống

## 2.1. Đối tượng sử dụng

- **Quản trị viên:** cấu hình hệ thống, luật và người dùng.
- **Nhân viên giám sát:** theo dõi cảnh báo, lọc sự kiện và xem chi tiết.
- **Người nghiên cứu:** xuất dữ liệu và đánh giá thuật toán.

## 2.2. Yêu cầu chức năng

| Mã | Yêu cầu |
|---|---|
| FR-01 | Thu thập cảnh báo từ Snort hoặc nguồn log tương thích. |
| FR-02 | Chuẩn hóa và lưu sự kiện mạng. |
| FR-03 | Hiển thị danh sách cảnh báo theo thời gian thực hoặc gần thời gian thực. |
| FR-04 | Lọc theo thời gian, IP, giao thức, mức độ và loại cảnh báo. |
| FR-05 | Tính điểm rủi ro cho sự kiện. |
| FR-06 | Phát hiện mẫu bất thường từ dữ liệu lưu lượng bằng thuật toán thống kê và luật ngưỡng. |
| FR-07 | Thống kê số lượng cảnh báo, nguồn tấn công và xu hướng. |
| FR-08 | Xuất dữ liệu phục vụ phân tích và báo cáo. |
| FR-09 | Nhận flow features từ flow extractor (Zeek). |
| FR-10 | Hợp nhất cảnh báo signature và kết quả phân tích bất thường thống kê thành một sự kiện tương quan. |
| FR-11 | Gửi cảnh báo theo mức độ nghiêm trọng qua Alertmanager. |
| FR-12 | Kiểm thử tự động rule Snort và thuật toán phát hiện trong pipeline CI/CD. |
| FR-13 | Triển khai các thành phần bằng Helm trên Kubernetes. |

## 2.3. Yêu cầu phi chức năng

- **Hiệu năng:** tiếp nhận được lưu lượng cảnh báo của môi trường lab mà không mất sự kiện đáng kể.
- **Tin cậy:** lỗi một sự kiện không làm dừng toàn bộ pipeline.
- **Bảo mật:** giới hạn quyền truy cập và bảo vệ endpoint tiếp nhận log.
- **Khả năng mở rộng:** có thể mở rộng quy mô các thành phần xử lý trên Kubernetes.
- **Khả năng kiểm thử:** mọi module chính có dữ liệu và kịch bản kiểm thử riêng.
- **Vận hành:** có metric cho Snort, pipeline xử lý và Anomaly Detection Service.
- **Triển khai:** cấu hình được quản lý dạng mã nguồn, có thể phục hồi và tái triển khai.

## 2.4. Dữ liệu sự kiện tối thiểu

Thời gian, IP nguồn, IP đích, cổng nguồn, cổng đích, giao thức, chữ ký Snort, loại cảnh báo, mức độ, số gói tin và số byte.

Flow features bổ sung thời lượng, số byte/gói theo chiều đi và về, tốc độ, cờ TCP và các đặc trưng thống kê theo cửa sổ thời gian cần cho thuật toán phát hiện bất thường.

## 2.5. Giới hạn an toàn

Mọi thử nghiệm phải thực hiện trên mạng lab hoặc tài sản được ủy quyền. Không dùng hệ thống để quét hay khai thác mục tiêu bên ngoài phạm vi cho phép.
