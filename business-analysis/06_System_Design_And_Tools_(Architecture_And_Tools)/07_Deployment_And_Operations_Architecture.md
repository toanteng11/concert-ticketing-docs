# 07 — Kiến Trúc Triển Khai & Vận Hành Hệ Thống (Deployment & Operations Architecture)

## 1. Giới Thiệu & Triết Lý Vận Hành

Tài liệu này đặc tả toàn bộ kiến trúc hạ tầng mạng, cấu trúc cụm máy chủ, quy trình phát hành phần mềm liên tục (**CI/CD**) và quy trình chuẩn bị vận hành trước giờ mở bán vé sự kiện (**Flash-Sale Operational Runbook**).

> **Ghi chú triển khai:** Nhờ chuyển sang kiến trúc **Modular Monolith**, cấu trúc triển khai đã được đơn giản hóa tối đa: từ 10+ Docker images (Microservices) xuống còn **3 services trong Docker Compose** (app + Redis + MySQL), giúp nhóm phát triển có thể chạy toàn bộ hệ thống trên laptop trong 30 giây.

Mục tiêu vận hành:
1. **Triển Khai Đơn Giản (Simple & Reproducible Deployment):** Toàn bộ stack triển khai được định nghĩa trong `docker-compose.yml` và `Dockerfile`, phiên bản hóa trong GitHub.
2. **Triển Khai Không Gián Đoạn (Zero-Downtime Deployment):** Nâng cấp phiên bản phần mềm mà không làm rớt kết nối hoặc gây lỗi giao dịch, thực hiện qua Rolling Update.
3. **Môi Trường Nhất Quán (Environment Parity):** Dev, Staging và Production dùng cùng Docker image, chỉ khác nhau ở file `.env` cấu hình.

---

## 2. Cấu Trúc Docker Compose (Development & Staging)

Môi trường phát triển và staging được cấu hình bằng Docker Compose với 3 services chính:

```yaml
# docker-compose.yml
version: '3.8'
services:

  app:
    image: ticketplatform/app:${APP_VERSION}
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - DB_HOST=mysql
      - DB_PORT=3306
      - DB_NAME=ticketdb
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - ticketing-network

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: ticketdb
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5
    networks:
      - ticketing-network

  redis:
    image: redis:7.2-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    networks:
      - ticketing-network

volumes:
  mysql-data:
  redis-data:

networks:
  ticketing-network:
    driver: bridge
```

---

## 3. Dockerfile Cho Ứng Dụng Spring Boot

```dockerfile
# Dockerfile — Multi-stage build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/target/ticket-platform-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", \
  "-XX:+UseG1GC", \
  "-Xms512m", "-Xmx2g", \
  "-Dspring.profiles.active=${SPRING_PROFILES_ACTIVE}", \
  "app.jar"]
```

---

## 4. Cấu Trúc Mạng & Phân Vùng (VPC & Subnet Topology — Production)

Môi trường Production được triển khai trên Cloud (AWS / GCP) với phân vùng mạng bảo mật:

| Phân vùng mạng | Dải IP (CIDR) | Truy cập Internet | Thành phần triển khai | Bảo vệ |
|---|---|---|---|---|
| **Public Subnets** | `10.0.1.0/24–3.0/24` | Trực tiếp qua IGW | Application Load Balancer (ALB), NAT Gateways | Chỉ mở TCP 443 & 80 |
| **Private App Subnets** | `10.0.10.0/22–18.0/22` | Outbound qua NAT | **Spring Boot App Pods** (EKS / Docker Swarm), Kong Gateway | Không có IP công khai |
| **Isolated DB Subnets** | `10.0.30.0/24–32.0/24` | **KHÔNG** | MySQL Primary & Replicas, Redis Cluster | Chỉ chấp nhận kết nối từ App Subnet |

---

## 5. Quy Trình CI/CD (GitHub Actions)

Toàn bộ chu trình từ khi developer commit đến khi chạy trên Production:

```
[Developer: git push → GitHub]
        ↓ (Kích hoạt GitHub Actions)
[1. Compile & Unit Tests (Maven / JUnit 5)]
   → Coverage >= 80% (JaCoCo)
        ↓
[2. Code Quality & Security Scan]
   → SonarQube Quality Gate
   → Trivy Container Vulnerability Scan
        ↓
[3. Build Docker Image]
   → docker build (multi-stage)
   → docker push → Docker Hub / AWS ECR
        ↓
[4. Deploy to Staging]
   → docker-compose up -d (staging env)
   → Run Integration Tests (Postman / RestAssured)
        ↓
[5. Manual Approval Gate]
   → Tech Lead review trước khi deploy Production
        ↓
[6. Deploy to Production]
   → Rolling Update (zero-downtime):
     Khởi động instance mới → Health check pass
     → Chuyển traffic → Tắt instance cũ
```

### Chiến Lược Rolling Update (Zero-Downtime)

Khi deploy phiên bản mới cho ứng dụng Monolith trong Kubernetes hoặc Docker Swarm:
- Khởi động Pod/Container mới với image version mới.
- Health check endpoint `/actuator/health` phải trả về `200 OK` trước khi nhận traffic.
- Load Balancer chuyển traffic sang instance mới từ từ.
- Tắt instance cũ sau khi tất cả request đang xử lý hoàn tất (Graceful Shutdown).

---

## 6. Cấu Hình Môi Trường (Environment Variables)

| Biến môi trường | Mô tả | Ví dụ |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Profile Spring Boot | `prod`, `staging`, `dev` |
| `DB_HOST` | MySQL host | `mysql-primary.internal` |
| `DB_PORT` | MySQL port | `3306` |
| `DB_NAME` | Tên database | `ticketdb` |
| `DB_USERNAME` | MySQL user | `appuser` |
| `DB_PASSWORD` | MySQL password | *(secret)* |
| `REDIS_HOST` | Redis host | `redis-cluster.internal` |
| `REDIS_PASSWORD` | Redis auth | *(secret)* |
| `JWT_SECRET_KEY` | JWT RS256 private key path | `/secrets/jwt-private.pem` |
| `VNPAY_TMN_CODE` | VNPay terminal code | *(secret)* |
| `VNPAY_HASH_SECRET` | VNPay hash secret | *(secret)* |
| `MAIL_HOST` | SMTP server | `smtp.sendgrid.net` |
| `MAIL_USERNAME` | SMTP user | *(secret)* |
| `MAIL_PASSWORD` | SMTP password | *(secret)* |

---

## 7. Quy Trình Vận Hành Trước Giờ Mở Bán Sự Kiện Lớn (Flash-Sale Runbook)

Quy trình bắt buộc thực thi trước mỗi sự kiện bán vé quy mô trên 10.000 chỗ ngồi:

| Mốc thời gian | Đội ngũ phụ trách | Hành động vận hành bắt buộc | Mục tiêu |
|---|---|---|---|
| **T - 2 Giờ** | Lead Backend & DevOps | 1. Kiểm tra sức khỏe tổng thể app, MySQL, Redis<br>2. Xác minh dung lượng trống MySQL > 40%<br>3. Kiểm tra hạn mức kết nối cổng VNPay/MoMo | Đảm bảo hạ tầng sạch |
| **T - 60 Phút** | Lead Backend | 1. **Feature Freeze** — đóng băng deploy mã mới<br>2. Scale-out: Tăng số lượng app instances lên tối đa<br>3. Pre-load concert & seat map vào Redis Cache | Tránh cold-start khi người dùng ùa vào |
| **T - 15 Phút** | Hệ thống tự động | 1. Kích hoạt Phòng Chờ Ảo (Waiting Room)<br>2. Chuyển hướng traffic vào queue endpoint<br>3. Bật chế độ ghi log tối giản (Minimal Logging) | Hấp thụ cú sốc lưu lượng ban đầu |
| **T (Giờ G)** | Hệ thống tự động | 1. Thuật toán xáo trộn số thứ tự hàng chờ<br>2. Mở van xả 5.000 CCU vào khu vực chọn ghế<br>3. Dev team túc trực monitor P99 Latency & Error Rate | Vận hành bán vé chính thức |
| **T + 30 Phút** | Product Owner & Lead Dev | 1. Vé bán hết hoặc lưu lượng hạ nhiệt: Đóng hàng chờ<br>2. Scale-in app instances về mức bình thường<br>3. Đối soát nhanh số vé đã bán vs. số tiền nhận | Hoàn tất đợt mở bán |

---

## 8. Giám Sát & Quan Sát Hệ Thống (Observability Stack)

| Công cụ | Vai trò | Tích hợp |
|---|---|---|
| **Spring Boot Actuator** | Health check, Metrics endpoint (`/actuator/health`, `/actuator/metrics`) | Tích hợp sẵn trong app |
| **Prometheus** | Thu thập metrics từ Actuator endpoint | Cấu hình scrape job trong prometheus.yml |
| **Grafana** | Dashboard hiển thị P50/P95/P99 Latency, Error Rate, JVM Heap Usage | Import dashboard từ Grafana Community |
| **Logback + ELK** | Structured JSON logging → Elasticsearch → Kibana | Cấu hình Logback appender |
| **Sentry** | Theo dõi Exception & Error Tracking real-time | Tích hợp Sentry SDK cho Spring Boot |
