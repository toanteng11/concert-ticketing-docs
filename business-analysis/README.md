# 🎫 Bộ Tài Liệu Phân Tích Nghiệp Vụ Hệ Thống Bán Vé Ca Nhạc Trực Tuyến (Concert Ticketing Business Analysis Documentation)

> **Mục tiêu tối thượng:** Bộ tài liệu Phân tích Nghiệp vụ (Business Analysis - BA) này được xây dựng chuẩn hóa, toàn diện và chặt chẽ, đóng vai trò là **cơ sở dữ liệu đặc tả tiền đề** để các kỹ sư hệ thống, kiến trúc sư phần mềm (Solution Architects), lập trình viên và kiểm thử viên có thể đọc hiểu và trực tiếp vẽ các sơ đồ kỹ thuật:
> - **Sơ đồ Tổng thể Kiến trúc (System Architecture Diagrams)**
> - **Sơ đồ Trường hợp Sử dụng (Use Case Diagrams - UML)**
> - **Sơ đồ Trình tự (Sequence Diagrams)**
> - **Sơ đồ Máy Trạng thái (State Machine Diagrams)**
> - **Sơ đồ Luồng Hoạt động (Activity / Flow Diagrams)**
> - **Sơ đồ Thực thể Liên kết (Entity Relationship Diagram - ERD)**
> - **Tài liệu Hợp đồng Giao diện Lập trình (API & Webhook Contracts)**

---

## 📑 Bảng Mục Lục & Cấu Trúc Toàn Bộ Dự Án

Toàn bộ hệ thống tài liệu được tổ chức thành **8 phân hệ nghiệp vụ chính (57 tài liệu Markdown, tính cả README)** theo thứ tự phát triển chuẩn công nghiệp:

```
business-analysis/
├── 00_Project_Governance/                           # Quản trị dự án & tiêu chuẩn tài liệu
│   ├── 01_Document_Changelog.md
│   ├── 02_Document_Conventions_And_Standards.md
│   └── 03_Stakeholder_Directory_And_RACI_Matrix.md
│
├── 01_Business_Overview/                               # Tổng quan nghiệp vụ, bối cảnh & vai trò
│   ├── 01_Business_Context_And_System_Scope.md
│   ├── 02_Actor_Permissions.md
│   ├── 03_Business_Rules_Catalog_(Business_Rules).md
│   └── 04_Glossary.md
│
├── 02_State_Machines_And_Workflows/                    # Máy trạng thái & luồng nghiệp vụ lõi
│   ├── SM01_Concert_State.md
│   ├── SM02_Seat_State.md
│   ├── SM03_Order_State.md
│   ├── SM04_Ticket_State.md
│   ├── SM05_Payment_And_Refund_State.md
│   ├── BF01_Waiting_Room_And_Virtual_Queue_Flow.md
│   ├── BF02_Seat_Selection_And_Double_Booking_Prevention.md
│   ├── BF03_VNPay_And_MoMo_Payment_Flow.md
│   ├── BF04_Order_Cancellation_And_Refund_Policy.md
│   ├── BF05_Concert_Approval_And_Cancellation_Flow.md
│   └── BF06_QR_Scanning_And_Gate_Check_In_Flow.md
│
├── 03_Functional_Requirements/                         # Yêu cầu chức năng chi tiết (15 Use Cases)
│   ├── 01_Account_Module/
│   │   ├── UC01_Registration_And_Login.md
│   │   ├── UC02_Personal_Account_Management.md
│   │   └── UC03_Admin_Account_Management.md
│   ├── 02_Concert_Ticket_Purchase_Module/
│   │   ├── UC04_Browse_Search_And_Filter_Concerts.md
│   │   ├── UC05_View_Seat_Map.md
│   │   ├── UC06_Select_And_Hold_Seats.md
│   │   └── UC07_Order_Checkout_And_Payment.md
│   ├── 03_Orders_Refunds_And_Reviews_Module/
│   │   ├── UC08_View_Personal_Order_History.md
│   │   ├── UC09_Order_Cancellation_And_Refund_Request.md
│   │   └── UC10_Concert_Reviews_And_Ratings.md
│   ├── 04_Organizer_And_Concert_Administration_Module/
│   │   ├── UC11_Create_Concert_(Organizer).md
│   │   ├── UC12_Concert_Approval_And_Cancellation_Management.md
│   │   └── UC13_Revenue_And_Ticket_Sales_Reports.md
│   ├── 05_Check_In_And_Notifications_Module/
│   │   ├── UC14_QR_Code_Check_In_(Staff).md
│   │   └── UC15_Notification_System.md
│   └── 06_Requirements_Traceability_Matrix.md          # Ma trận truy vết RTM
│
├── 04_Capacity_Planning_And_Resource_Estimation/       # Ước tính quy mô tải, lưu trữ & tài nguyên
│   ├── 01_Traffic_And_Throughput_Estimation_(QPS_CCU).md
│   ├── 02_Storage_Capacity_Estimation.md
│   └── 03_Bandwidth_CDN_And_Server_Estimation.md
│
├── 05_Non_Functional_Requirements_(NFR)/               # Yêu cầu phi chức năng cấp doanh nghiệp
│   ├── 01_Cross_Platform_Caching_And_CDN.md
│   ├── 02_Handling_100K_Concurrent_Users.md
│   ├── 03_Double_Booking_And_Duplicate_Payment_Prevention.md
│   ├── 04_Concurrency_And_Failure_Scenarios_(CAP_Theorem).md
│   ├── 05_Database_Optimization_And_System_Security.md
│   ├── 06_Performance_And_Availability_Targets.md
│   ├── 07_Monitoring_Backup_And_Disaster_Recovery.md
│   └── 08_Data_Privacy_Audit_And_Retention.md
│
├── 06_System_Design_And_Tools_(Architecture_And_Tools)/ # Thiết kế kiến trúc, ERD & API
│   ├── 01_High_Level_Client_Server_And_API_Gateway_Architecture.md
│   ├── 02_Modular Monolith_Architecture_And_Design.md
│   ├── 03_ERD_And_Database_Schema_Design.md
│   ├── 04_RESTful_API_And_Webhook_Design.md
│   ├── 05_Technology_Stack_And_Development_Tools.md
│   ├── 06_External_Integration_Contracts.md
│   └── 07_Deployment_And_Operations_Architecture.md
│
└── 07_Validation_And_Testing/                          # Tiêu chí nghiệm thu & kiểm thử
    ├── 01_Acceptance_Criteria_And_UAT.md
    ├── 02_Functional_And_Edge_Case_Test_Scenarios.md
    └── 03_Performance_And_Resilience_Test_Plan.md
```

---

## 🛠️ Bộ Công Cụ & Ngăn Xếp Công Nghệ Chuẩn Hóa

Dự án thống nhất sử dụng bộ công cụ và công nghệ chuẩn hóa sau cho toàn bộ các giai đoạn phân tích, thiết kế và phát triển:

| Vai trò / Lĩnh vực | Công cụ / Công nghệ chuẩn hóa | Phiên bản & Mục đích sử dụng |
|---|---|---|
| **Ngôn ngữ Backend chính** | **Java** | **Java 17 / Java 21 LTS**, nền tảng **Spring Boot 3** (Spring Data JPA, Spring Security) |
| **Công cụ Lập trình (IDE)** | **Visual Studio Code (VS Code)** | Bộ tiện ích Extension Pack for Java, Spring Boot, Docker, GitLens, SonarLint |
| **Cơ Sở Dữ Liệu (RDBMS)** | **Microsoft SQL Server 2025 Enterprise** | Production Always On AG theo D-026; local development dùng Developer edition; DBA/Finance xác nhận license và sizing trước go-live |
| **Thiết kế Giao diện (UI/UX)** | **Figma** | Thiết kế Design System, Wireframes, Mockup & Prototype chọn ghế trực quan |
| **Thiết kế Sơ đồ Kỹ thuật (UML)** | **Astah (Astah Professional / UML)**| Vẽ toàn bộ sơ đồ Use Case, Sequence, State Machine, Class, Component Diagram |
| **Kiểm thử Giao diện Lập trình** | **Postman** | Postman Collections, biến môi trường, Pre-request Scripts & Test Assertions tự động |
| **Quản lý Tiến độ Dự án (Agile)**| **Jira (Jira Software)** | Quản lý Sprint, Epics, User Stories (`US-01-01` đến `US-15-03`), Burndown Chart |
| **Quản lý Mã nguồn (VCS)** | **GitHub** | Lưu trữ mã nguồn, quy trình GitFlow, Pull Request, Code Review, GitHub Actions CI/CD |
| **Đóng gói & Môi trường Chạy** | **Docker** | Multi-stage Dockerfile cho Java Spring Boot; Compose local dùng SQL Server 2025 Developer, Redis và RabbitMQ; không có NoSQL trong baseline |

---

## 🗺️ Hướng Dẫn Sử Dụng Tài Liệu Để Vẽ Các Sơ Đồ Thiết Kế (Bằng Astah & Figma)

Dành cho các thành viên đội ngũ thiết kế, đồ họa kỹ thuật và kiến trúc sư hệ thống khi thao tác trên **Astah** và **Figma**:

| Sơ đồ cần vẽ trên Astah / Figma | Tài liệu nguồn cần đọc | Dữ liệu đầu vào then chốt |
|---|---|---|
| **Use Case Diagram (Astah)** | `UC01` đến `UC15` + `02_Actor_Permissions.md` | Bảng phân quyền RBAC, danh sách Use Case con, quan hệ `<<include>>` và `<<extend>>`. |
| **Sequence Diagrams (Astah)** | `BF02`, `BF03`, `BF06`, `04_RESTful_API...` | Luồng tương tác tuần tự giữa Client $\rightarrow$ Gateway $\rightarrow$ Java Service $\rightarrow$ Redis $\rightarrow$ SQL Server $\rightarrow$ Third-party. |
| **State Machine Diagrams (Astah)** | `SM01` đến `SM05` | 5 vòng đời: Concert, Ghế (Seat), Đơn hàng (Order), Vé (Ticket) và Thanh toán/Hoàn tiền. |
| **Activity / Flowcharts (Astah)** | `BF01` đến `BF06` | Các bước phân nhánh điều kiện: Hàng chờ, Giữ ghế, Thanh toán, Hủy/hoàn tiền, Duyệt sự kiện, Quét QR. |
| **Entity Relationship Diagram (Astah)** | 03_ERD_And_Database_Schema_Design.md | 20 bảng nghiệp vụ + transactional outbox (21 bảng); mô hình logic Microsoft SQL Server, PK/FK và ràng buộc. |
| **High-Level System Architecture (Astah)**| `01_High_Level_Architecture...` | Client → CDN/WAF → Gateway → Modular Monolith + Worker; Redis, RabbitMQ, SQL Server Primary/readable secondary + async DR replica; không có NoSQL. |
| **UI/UX Wireframes & Prototypes (Figma)** | `UC04`, `UC05`, `UC06`, `UC07`, `UC14` | Giao diện phòng chờ ảo, sơ đồ ghế khán đài SVG trực quan, màn hình thanh toán, mã Dynamic QR. |

---

## 🌟 Các Chỉ Số Kỹ Thuật Đạt Chuẩn Của Hệ Thống

- **Quy mô người dùng đồng thời đỉnh (Flash-Sale):** $100.000\text{ CCU}$
- **Thông lượng tiếp nhận API Gateway:** $\ge 35.000\text{ QPS}$
- **SLO khả dụng:** 99.95% mỗi tháng cho critical first-party purchase APIs theo D-027; payment provider được đo riêng; không tuyên bố SLA hợp đồng.
- **Thời gian giữ ghế tạm thời:** Đúng $600\text{ giây}$ ($10\text{ phút}$)
- **Tốc độ phản hồi quét vé tại cổng:** $\le 500\text{ ms}$
- **Nguyên tắc cốt lõi:** **Tuyệt đối không bán trùng ghế (Zero Double-Booking)** & **Chống thanh toán trùng lặp (Idempotent Payment)**.




## Cách đọc và xử lý điểm chưa thống nhất

Đọc [Chuẩn biên tập và cách đọc bộ tài liệu BA](./00_Project_Governance/04_BA_Editorial_And_Consistency_Standards.md) trước khi dùng các tài liệu chi tiết. Câu hỏi chưa được xác nhận được theo dõi trong [Decision Log](./00_Project_Governance/03_Risks_Open_Questions_And_Decision_Log.md); không xem đề xuất kỹ thuật là quyết định nghiệp vụ.
