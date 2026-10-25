# 🎫 Concert Ticketing Platform — Business Analysis Documentation

Chào mừng bạn đến với kho tài liệu **Phân Tích Nghiệp Vụ (Business Analysis - BA)** toàn diện cho **Hệ Thống Bán Vé Ca Nhạc Trực Tuyến (Concert Ticketing Platform)**.

Kho tài liệu này mô tả phạm vi nghiệp vụ, quy tắc, luồng, yêu cầu phi chức năng và phương án thiết kế dự kiến cho nền tảng bán vé concert. Các quyết định đã chốt được ghi trong Decision Log; các giả định và câu hỏi chưa chốt cần được xác nhận trước khi dùng làm cam kết triển khai.

---

## 📑 Truy Cập Tài Liệu Chi Tiết

👉 **[Xem Tài Liệu Phân Tích Nghiệp Vụ Tổng Thể (Master BA Documentation)](business-analysis/README.md)**

Trước khi sử dụng tài liệu chi tiết, xem [chuẩn biên tập và cách đọc bộ BA](business-analysis/00_Project_Governance/04_BA_Editorial_And_Consistency_Standards.md) cùng [các câu hỏi mở và quyết định](business-analysis/00_Project_Governance/03_Risks_Open_Questions_And_Decision_Log.md).

---

## 🛠️ Bộ Công Nghệ & Công Cụ Chuẩn Hóa Dự Án

| Vai trò / Lĩnh vực | Công cụ / Công nghệ chuẩn hóa | Phiên bản & Mục đích sử dụng |
|---|---|---|
| **Ngôn ngữ Backend chính** | **Java** | **Java 17 / Java 21 LTS**, nền tảng **Spring Boot 3** (Spring Data JPA, Spring Security) |
| **Công cụ Lập trình (IDE)** | **Visual Studio Code (VS Code)** | Bộ tiện ích Extension Pack for Java, Spring Boot, Docker, GitLens, SonarLint |
| **Cơ Sở Dữ Liệu (RDBMS)** | **Microsoft SQL Server** | Phiên bản/edition và HA đang chờ Q-013; giao dịch ACID, lock hints trong transaction |
| **Thiết kế Giao diện (UI/UX)** | **Figma** | Thiết kế Design System, Wireframes, Mockup & Prototype chọn ghế trực quan |
| **Thiết kế Sơ đồ Kỹ thuật (UML)** | **Astah (Astah Professional / UML)**| Vẽ toàn bộ sơ đồ Use Case, Sequence, State Machine, Class, Component Diagram |
| **Kiểm thử Giao diện Lập trình** | **Postman** | Postman Collections, biến môi trường, Pre-request Scripts & Test Assertions tự động |
| **Quản lý Tiến độ Dự án (Agile)**| **Jira (Jira Software)** | Quản lý Sprint, Epics, User Stories (`US-01-01` đến `US-15-03`), Burndown Chart |
| **Quản lý Mã nguồn (VCS)** | **GitHub** | Lưu trữ mã nguồn, quy trình GitFlow, Pull Request, Code Review, GitHub Actions CI/CD |
| **Đóng gói & Môi trường Chạy** | **Docker** | Multi-stage Dockerfile cho Java Spring Boot, `docker-compose.yml` chạy SQL Server, Redis và broker (Q-012); NoSQL event/log store là thành phần phụ (Q-016) |

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
| **Sequence Diagrams (Astah)** | `BF02`, `BF03`, `BF06`, `04_RESTful_API...` | Luồng tương tác tuần tự giữa Client $\rightarrow$ Gateway $\rightarrow$ Java Service $\rightarrow$ Redis $\rightarrow$ SQL Server $\rightarrow$ Third-party. |
| **State Machine Diagrams (Astah)** | `SM01` đến `SM05` | 5 vòng đời: Concert, Ghế (Seat), Đơn hàng (Order), Vé (Ticket) và Thanh toán/Hoàn tiền. |
| **Activity / Flowcharts (Astah)** | `BF01` đến `BF06` | Các bước phân nhánh điều kiện: Hàng chờ, Giữ ghế, Thanh toán, Hủy/hoàn tiền, Duyệt sự kiện, Quét QR. |
| **Entity Relationship Diagram (Astah)** | `03_ERD_And_Database_Schema_Design.md` | 19 thực thể chuẩn SQL Server, kiểu dữ liệu, khóa PK/FK, chỉ mục B-tree và ràng buộc toàn vẹn. |
| **High-Level System Architecture (Astah)**| `01_High_Level_Architecture...` | Kiến trúc tổng thể: Client → CDN/WAF → Gateway → Modular Monolith + Worker; Redis, broker (Q-012), SQL Server Primary/readable secondary và NoSQL phụ (Q-016). |
| **UI/UX Wireframes & Prototypes (Figma)** | `UC04`, `UC05`, `UC06`, `UC07`, `UC14` | Giao diện phòng chờ ảo, sơ đồ ghế khán đài SVG trực quan, màn hình thanh toán, mã Dynamic QR. |

---

## 🌟 Cam Kết Chỉ Số Kỹ Thuật

- **Quy mô đỉnh tải:** $100.000\text{ CCU}$
- **Thông lượng API Gateway:** $\ge 35.000\text{ QPS}$
- **Độ sẵn sàng (mục tiêu):** 99.95% trong cửa sổ mở bán; phạm vi đo và SLO kỹ thuật 99.99% cần xác nhận tại Q-014.
- **Thời gian giữ ghế tạm thời:** Đúng $600\text{ giây}$ ($10\text{ phút}$)
- **Tốc độ phản hồi quét vé tại cổng:** $\le 500\text{ ms}$
- **Nguyên tắc vàng:** **Tuyệt đối không bán trùng ghế (Zero Double-Booking)** & **Chống thanh toán trùng lặp (Idempotent Payment)**.


