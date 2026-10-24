# 🎫 Concert Ticketing Platform — Business Analysis Documentation

Chào mừng bạn đến với kho tài liệu **Phân Tích Nghiệp Vụ (Business Analysis - BA)** toàn diện cho **Hệ Thống Bán Vé Ca Nhạc Trực Tuyến (Concert Ticketing Platform)**.

Kho tài liệu này được xây dựng chuẩn hóa, chặt chẽ và đầy đủ số liệu kỹ thuật, đóng vai trò là **tiền đề và tài liệu đặc tả** để các kỹ sư hệ thống, kiến trúc sư phần mềm (Solution Architects), lập trình viên và kiểm thử viên sử dụng trực tiếp để thiết kế, vẽ sơ đồ và cài đặt hệ thống.

---

## 📑 Truy Cập Tài Liệu Chi Tiết

👉 **[Xem Tài Liệu Phân Tích Nghiệp Vụ Tổng Thể (Master BA Documentation)](business-analysis/README.md)**

---

## 🛠️ Bộ Công Nghệ & Công Cụ Chuẩn Hóa Dự Án

| Vai trò / Lĩnh vực | Công cụ / Công nghệ chuẩn hóa | Phiên bản & Mục đích sử dụng |
|---|---|---|
| **Ngôn ngữ Backend chính** | **Java** | **Java 17 / Java 21 LTS**, nền tảng **Spring Boot 3** (Spring Data JPA, Spring Security) |
| **Công cụ Lập trình (IDE)** | **Visual Studio Code (VS Code)** | Bộ tiện ích Extension Pack for Java, Spring Boot, Docker, GitLens, SonarLint |
| **Cơ Sở Dữ Liệu (RDBMS)** | **MySQL** | **MySQL 8.0+ (InnoDB Engine)**, giao dịch ACID, khóa cấp dòng `FOR UPDATE` |
| **Thiết kế Giao diện (UI/UX)** | **Figma** | Thiết kế Design System, Wireframes, Mockup & Prototype chọn ghế trực quan |
| **Thiết kế Sơ đồ Kỹ thuật (UML)** | **Astah (Astah Professional / UML)**| Vẽ toàn bộ sơ đồ Use Case, Sequence, State Machine, Class, Component Diagram |
| **Kiểm thử Giao diện Lập trình** | **Postman** | Postman Collections, biến môi trường, Pre-request Scripts & Test Assertions tự động |
| **Quản lý Tiến độ Dự án (Agile)**| **Jira (Jira Software)** | Quản lý Sprint, Epics, User Stories (`US-01-01` đến `US-15-03`), Burndown Chart |
| **Quản lý Mã nguồn (VCS)** | **GitHub** | Lưu trữ mã nguồn, quy trình GitFlow, Pull Request, Code Review, GitHub Actions CI/CD |
| **Đóng gói & Môi trường Chạy** | **Docker** | Multi-stage Dockerfile cho Java Spring Boot, `docker-compose.yml` chạy MySQL, Redis, Kafka |

---

## 📁 Cấu Trúc Thư Mục Tài Liệu (`business-analysis/`)

```
business-analysis/
├── 00_Project_Governance/                           # Quản trị dự án, các bên liên quan & RACI
├── 01_Business_Overview/                               # Bối cảnh kinh doanh, RBAC & Quy tắc nghiệp vụ (BR)
├── 02_State_Machines_And_Workflows/                    # 5 Cỗ máy trạng thái & 6 Luồng nghiệp vụ cốt lõi
├── 03_Functional_Requirements/                         # 15 Use Cases chi tiết & Ma trận truy vết RTM
├── 04_Capacity_Planning_And_Resource_Estimation/       # Ước tính QPS, 100K CCU, Dung lượng DB & Băng thông
├── 05_Non_Functional_Requirements_(NFR)/               # Yêu cầu phi chức năng (Chống bán trùng, SLA, Bảo mật)
├── 06_System_Design_And_Tools_(Architecture_And_Tools)/ # Thiết kế 7 tầng, Modular Monolith DDD, ERD 19 bảng, API
└── 07_Validation_And_Testing/                          # Tiêu chí nghiệm thu UAT, 15 ca chức năng & 15 ca biên
```

---

## 🗺️ Hướng Dẫn Vẽ Sơ Đồ Thiết Kế Bằng Astah & Figma

| Sơ đồ cần vẽ trên Astah / Figma | Tài liệu nguồn cần đọc | Dữ liệu đầu vào then chốt |
|---|---|---|
| **Use Case Diagram (Astah)** | `UC01` đến `UC15` + `02_Actor_Permissions.md` | Bảng phân quyền RBAC, danh sách Use Case con, quan hệ `<<include>>` và `<<extend>>`. |
| **Sequence Diagrams (Astah)** | `BF02`, `BF03`, `BF06`, `04_RESTful_API...` | Luồng tương tác tuần tự giữa Client $\rightarrow$ Gateway $\rightarrow$ Java Service $\rightarrow$ Redis $\rightarrow$ MySQL $\rightarrow$ Third-party. |
| **State Machine Diagrams (Astah)** | `SM01` đến `SM05` | 5 vòng đời: Concert, Ghế (Seat), Đơn hàng (Order), Vé (Ticket) và Thanh toán/Hoàn tiền. |
| **Activity / Flowcharts (Astah)** | `BF01` đến `BF06` | Các bước phân nhánh điều kiện: Hàng chờ, Giữ ghế, Thanh toán, Hủy/hoàn tiền, Duyệt sự kiện, Quét QR. |
| **Entity Relationship Diagram (Astah)** | `03_ERD_And_Database_Schema_Design.md` | 19 thực thể chuẩn MySQL 8.0, kiểu dữ liệu, khóa PK/FK, chỉ mục B-tree và ràng buộc toàn vẹn. |
| **High-Level System Architecture (Astah)**| `01_High_Level_Architecture...` | 7 tầng kiến trúc: Client $\rightarrow$ CDN/WAF $\rightarrow$ Gateway $\rightarrow$ Modular Monolith $\rightarrow$ Cache $\rightarrow$ Kafka $\rightarrow$ MySQL. |
| **UI/UX Wireframes & Prototypes (Figma)** | `UC04`, `UC05`, `UC06`, `UC07`, `UC14` | Giao diện phòng chờ ảo, sơ đồ ghế khán đài SVG trực quan, màn hình thanh toán, mã Dynamic QR. |

---

## 🌟 Cam Kết Chỉ Số Kỹ Thuật

- **Quy mô đỉnh tải:** $100.000\text{ CCU}$
- **Thông lượng API Gateway:** $\ge 35.000\text{ QPS}$
- **Độ sẵn sàng (Uptime SLA):** $99.99\%$ (Downtime $< 4.38\text{ phút/tháng}$)
- **Thời gian giữ ghế tạm thời:** Đúng $600\text{ giây}$ ($10\text{ phút}$)
- **Tốc độ phản hồi quét vé tại cổng:** $\le 500\text{ ms}$
- **Nguyên tắc vàng:** **Tuyệt đối không bán trùng ghế (Zero Double-Booking)** & **Chống thanh toán trùng lặp (Idempotent Payment)**.


