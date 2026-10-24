# 07 — Kiến Trúc Triển Khai & Vận Hành Hệ Thống (Deployment & Operations Architecture)

## 1. Giới Thiệu & Triết Lý Vận Hành

Tài liệu này đặc tả toàn bộ kiến trúc hạ tầng mạng, cấu trúc cụm máy chủ điều phối (**Kubernetes Cluster Topology**), quy trình phát hành phần mềm liên tục (**CI/CD & GitOps**) và quy trình chuẩn bị vận hành trước giờ mở bán vé sự kiện (**Flash-Sale Operational Runbook**).

Mục tiêu vận hành:
1. **Cô Lập Mạng Tuyệt Đối (Network Isolation & Zero Trust):** Các thành phần cơ sở dữ liệu và lưu trữ không có IP công khai, hoàn toàn được bảo vệ bên trong các phân vùng mạng riêng (Private/Isolated Subnets).
2. **Triển Khai Bất Biến (Immutable Infrastructure & GitOps):** Toàn bộ trạng thái hạ tầng và cấu hình ứng dụng được phiên bản hóa trong kho Git (ArgoCD), không cấu hình thủ công trên máy chủ sản xuất.
3. **Phát Hành Không Gián Đoạn (Zero-Downtime Deployment):** Nâng cấp phiên bản phần mềm trong khi hệ thống đang phục vụ khách hàng mà không làm rớt kết nối hoặc gây lỗi giao dịch.

---

## 2. Cấu Trúc Mạng & Phân Vùng Mạng Ảo (VPC & Subnet Topology)

Hệ thống được triển khai bên trong một Đám mây Riêng Ảo (**Virtual Private Cloud - VPC**) trải dài trên **3 Vùng Khả Dụng (Availability Zones - AZ1, AZ2, AZ3)**:

| Phân vùng mạng (Subnet Tier) | Dải địa chỉ IP (CIDR) | Khả năng truy cập Internet | Các thành phần triển khai bên trong | Cơ chế bảo vệ |
|---|---|---|---|---|
| **1. Public Subnets** (Mạng Công Khai) | `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24` | Trực tiếp qua Internet Gateway (IGW) | AWS Application Load Balancer (ALB), NAT Gateways | Chỉ mở cổng TCP 443 (HTTPS) và 80 (HTTP redirect); Chặn toàn bộ cổng quản trị |
| **2. Private Application Subnets** (Mạng Ứng Dụng) | `10.0.10.0/22`, `10.0.14.0/22`, `10.0.18.0/22` | Chiều ra (Outbound only) qua NAT Gateway | Cụm máy chủ Kubernetes EKS Worker Nodes, Kong Gateway, 10 Microservices | Không có IP công khai; Giao tiếp nội bộ qua mạng VPC ảo |
| **3. Isolated Database Subnets** (Mạng Dữ Liệu Cô Lập) | `10.0.30.0/24`, `10.0.31.0/24`, `10.0.32.0/24` | **HOÀN TOÀN KHÔNG** (No Internet) | PostgreSQL Primary & Replicas, Redis Cluster, Kafka Brokers | Cách ly tuyệt đối; Chỉ chấp nhận kết nối từ dải IP của Application Subnet qua Security Groups |

---

## 3. Kiến Trúc Cụm Kubernetes (EKS Cluster Architecture)

Cụm Kubernetes được quản lý bởi dịch vụ **AWS EKS** (hoặc Google GKE) với các nhóm máy chủ (Node Groups) phân theo chức năng:

### 1. Phân Bổ Nhóm Node Máy Chủ (Worker Node Groups)
- **Nhóm 1: Core Compute Node Group (General Purpose):**
  - Cấu hình: Thể hiện máy ảo `c6g.2xlarge` (8 vCPU, 16 GB RAM, chip ARM Graviton3).
  - Chạy các dịch vụ: API Gateway, Auth, Catalog, Booking, Order, Payment.
  - Chính sách co giãn: Tối thiểu 6 nodes, tối đa 24 nodes trong giờ mở bán cao điểm.
- **Nhóm 2: Memory-Optimized Node Group:**
  - Cấu hình: Thể hiện máy ảo `r6g.xlarge` (4 vCPU, 32 GB RAM).
  - Dành riêng cho các ứng dụng đệm và xếp hàng đòi hỏi băng thông RAM cực lớn.
- **Nhóm 3: Spot Instance Node Group (Tiết kiệm chi phí):**
  - Cấu hình: Máy ảo dạng Spot tiết kiệm $70\%$ chi phí.
  - Chạy các tác vụ xử lý nền không nhạy cảm với việc bị tắt đột ngột: Notification Worker, Async Excel Export Worker, ClickHouse Data Sync.

### 2. Phân Chia Không Gian Tên (Kubernetes Namespaces)
- `ingress-system`: Chứa Kong Gateway Ingress Controller và Cert-Manager.
- `ticketing-prod`: Toàn bộ 10 Microservices môi trường Production.
- `ticketing-staging`: Môi trường kiểm thử nghiệm thu (Pre-prod).
- `monitoring`: Prometheus, Grafana, Jaeger, FluentBit.
- `kafka`: Cụm Apache Kafka Strimzi Operator.

---

## 4. Quy Trình Tích Hợp & Triển Khai Tự Động (CI/CD Pipeline with GitOps)

Toàn bộ chu trình từ lúc lập trình viên commit mã nguồn đến khi chạy trên môi trường thực tế tuân thủ quy trình 6 bước tự động hóa $100\%$:

```
[Developer Git Commit]
       ↓ (1. Kích hoạt GitHub Actions)
[Linting, Compile & Unit Tests (Coverage >= 80%)]
       ↓ (2. Kiểm tra chất lượng & An ninh)
[SonarQube Quality Gate & Trivy Container Scan]
       ↓ (3. Đóng gói & Phát hành)
[Build Multi-stage Docker Image → Đẩy vào AWS ECR]
       ↓ (4. Cập nhật Manifest GitOps)
[Tự động tạo PR cập nhật Image Tag trong repo K8s-Configs]
       ↓ (5. Điều phối GitOps)
[ArgoCD phát hiện commit mới → Đồng bộ vào Cụm EKS]
       ↓ (6. Chiến lược phát hành an toàn)
[Canary Deployment: 5% → 25% → 100% sau 10 phút kiểm tra lỗi]
```

### Chiến Lược Phát Hành Hoàng Yến (Canary Deployment Strategy)
Đối với các dịch vụ lõi nhạy cảm như `Booking Service` và `Payment Service`:
- Phiên bản mới (Canary Pods) được triển khai song song và chỉ tiếp nhận **$5\%$** lưu lượng truy cập thực tế.
- Hệ thống Prometheus tự động giám sát tỷ lệ lỗi (Error Rate) và độ trễ P99 của phiên bản mới trong vòng **10 phút**.
- Nếu tỷ lệ lỗi tăng quá $0.1\%$ hoặc độ trễ tăng quá $20\%$: ArgoCD kích hoạt cơ chế **Tự Động Quay Đầu (Automated Rollback)** về phiên bản cũ ngay lập tức mà không cần can thiệp thủ công.
- Nếu các chỉ số ổn định: Lưu lượng được tăng dần lên $25\% \rightarrow 50\% \rightarrow 100\%$.

---

## 5. Quy Trình Vận Hành Trước Giờ Mở Bán Sự Kiện Lớn (Mega Flash-Sale Runbook)

Đây là quy trình bắt buộc phải thực thi trước mỗi sự kiện bán vé quy mô trên $10.000$ chỗ ngồi:

| Mốc thời gian | Đội ngũ phụ trách | Hành động vận hành bắt buộc | Mục tiêu kiểm soát |
|---|---|---|---|
| **$T - 2$ Giờ** | Lead SRE & DevOps | 1. Kiểm tra tình trạng sức khỏe tổng thể các cụm máy chủ (Node Health Checks)<br>2. Xác minh dung lượng trống ổ cứng SSD PostgreSQL và Kafka còn $> 40\%$<br>3. Kiểm tra hạn mức kết nối cổng thanh toán VNPay và MoMo | Đảm bảo hạ tầng sạch và không có sự cố tiềm ẩn |
| **$T - 60$ Phút**| SRE & Lead Backend | 1. Đóng băng toàn bộ hoạt động deploy mã nguồn mới (Feature Freeze)<br>2. Kích hoạt chế độ Warm-Up: Nâng số lượng Pods lên quy mô tối đa ($84\text{ pods}$)<br>3. Pre-load sơ đồ ghế và cấu hình concert vào Redis Cluster | Tránh hiện tượng trễ khởi động container khi người dùng ùa vào |
| **$T - 15$ Phút**| Hệ thống tự động | 1. Kích hoạt **Phòng Chờ Ảo (Holding Room)**<br>2. Chuyển hướng toàn bộ người truy cập mới vào phòng chờ gom lưu lượng<br>3. Bật chế độ ghi log tối giản (Minimal Logging) để tiết kiệm I/O | Hấp thụ cú sốc lưu lượng ban đầu |
| **$T$ (Giờ G)** | Hệ thống tự động | 1. Chạy thuật toán xáo trộn ngẫu nhiên phân tán số thứ tự hàng chờ<br>2. Mở van xả điều tiết $5.000\text{ CCU}$ vào khu vực chọn ghế<br>3. Đội ngũ kỹ thuật túc trực màn hình Dashboard giám sát P99 Latency và Error Rate | Vận hành bán vé chính thức |
| **$T + 30$ Phút**| Product Owner & SRE | 1. Vé đã bán hết hoặc lưu lượng hạ nhiệt: Đóng phòng chờ ảo<br>2. Hạ số lượng Pods về mức bình thường (Scale-Down Cooldown)<br>3. Tiến hành đối soát nhanh số vé đã bán và số tiền ghi nhận trong tài khoản Escrow | Hoàn tất đợt mở bán thành công |
