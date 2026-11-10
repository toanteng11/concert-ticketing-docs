# 07 — Kiến Trúc Triển Khai & Vận Hành Hệ Thống (Deployment & Operations Architecture)

## 1. Giới Thiệu & Triết Lý Vận Hành

Tài liệu này đặc tả kiến trúc hạ tầng, quy trình CI/CD và runbook vận hành Flash-Sale cho hệ thống Concert Ticketing.

> **Ghi chú triển khai:** Nhờ kiến trúc **Hybrid Modular Monolith + Worker**, cấu trúc triển khai cân bằng giữa đơn giản và khả năng scale: **6 services minh họa trong Docker Compose** (nginx + app + worker + rabbitmq + redis + SQL Server). So với Microservices thuần túy (10+ services), cấu trúc này dễ vận hành hơn nhiều trong khi vẫn có khả năng scale Worker độc lập.

---

## 2. Cấu Trúc Docker Compose (Development & Staging)

Đây là mẫu local development dùng SQL Server 2025 Developer và RabbitMQ. AWS target: EKS, Amazon MQ for RabbitMQ, ElastiCache Redis OSS và SQL Server 2025 Enterprise Always On AG trên EC2. Developer edition không dùng production.

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      app:
        condition: service_started
    networks: [ticketing-network]

  app:
    image: ticketplatform/monolith:${APP_VERSION}
    build:
      context: ./ticket-platform
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: ${ENV:-dev}
      DB_HOST: sqlserver
      DB_PORT: 1433
      DB_NAME: ticketdb
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      RABBITMQ_HOST: rabbitmq
    depends_on:
      sqlserver:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks: [ticketing-network]

  worker:
    image: ticketplatform/worker:${WORKER_VERSION}
    build:
      context: ./ticket-worker
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: ${ENV:-dev}
      DB_HOST: sqlserver
      DB_PORT: 1433
      DB_NAME: ticketdb
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      RABBITMQ_HOST: rabbitmq
      AWS_SES_CONFIGURATION_SET: configured-in-secrets-manager
      SNS_PLATFORM_APPLICATION_ARN: AWS-resource-arn
    depends_on:
      sqlserver:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks: [ticketing-network]

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USERNAME}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      retries: 5
    networks: [ticketing-network]

  redis:
    image: redis:7.2-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    environment:
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a \"$${REDIS_PASSWORD}\" ping | grep -q PONG"]
      interval: 5s
      retries: 5
    networks: [ticketing-network]

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2025-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: ${MSSQL_SA_PASSWORD}
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql
    healthcheck:
      test:
        - CMD-SHELL
        - >-
          /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$${MSSQL_SA_PASSWORD}"
          -C -Q "SELECT 1" || exit 1
      interval: 10s
      retries: 10
      start_period: 30s
    networks: [ticketing-network]

volumes:
  sqlserver-data:
  redis-data:
  rabbitmq-data:

networks:
  ticketing-network:
    driver: bridge
```

> SQL Server container example is for local development. Developer edition is not licensed for production. The image provisions the system administrator login; a migration/init task must create the database and least-privilege application login. Production uses SQL Server 2025 Enterprise Always On AG with synchronous local secondary and asynchronous cross-region DR per D-026. DBA/Finance must sign off license and sizing before go-live. NoSQL is excluded by D-029.

---

## 3. Dockerfile — Multi-Stage Build

### Monolith App

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests -pl ticket-platform

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/ticket-platform/target/ticket-platform-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseG1GC", "-Xms512m", "-Xmx2g", "-jar", "app.jar"]
```

### Worker Service

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests -pl ticket-worker

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/ticket-worker/target/ticket-worker-*.jar worker.jar
EXPOSE 8081
ENTRYPOINT ["java", "-XX:+UseG1GC", "-Xms256m", "-Xmx1g", "-jar", "worker.jar"]
```

---

## 4. Cấu Hình RabbitMQ — Queues & Dead-Letter

```
Exchange: notification.exchange (direct)
  ├── Queue: notification.send     → Worker Consumer (Email + Push)
  │     └── x-dead-letter-exchange: dlq.exchange
  │     └── x-message-ttl: 300000 (5 phút — nếu không xử lý được)
  ├── Queue: order.cancel          → Worker Consumer (Hủy đơn)
  └── Queue: refund.process        → Worker Consumer (Hoàn tiền VNPay/MoMo)

Exchange: dlq.exchange (fanout)
  └── Queue: notification.dlq      → Alert team kỹ thuật khi message thất bại 3 lần
```

---

## 5. Cấu Trúc Mạng Production (VPC & Subnet)

| Phân vùng mạng | Dải IP | Truy cập Internet | Thành phần | Bảo vệ |
|---|---|---|---|---|
| **Public subnets per AZ** | CIDR theo network plan | Internet Gateway | ALB, NAT Gateway per AZ | HTTPS ingress; origin restricted |
| **Private EKS subnets** | CIDR theo network plan | NAT có kiểm soát/VPC endpoints | EKS EC2 nodes, App/Worker Pods | Không public IP |
| **Private broker subnets** | CIDR theo network plan | No public ingress | Amazon MQ RabbitMQ multi-AZ | App/Worker SG via TLS |
| **Private data subnets** | CIDR theo network plan | No direct internet | SQL Server EC2 AG, ElastiCache Redis OSS | Workload/DBA access via approved SG/SSM |

<!-- BEGIN MERMAID DIAGRAM DEPLOY-VPC -->
### Topology mạng Production theo trust boundary

```mermaid
flowchart TB
  internet["Internet clients"] --> edge["CloudFront + AWS WAF<br/>DNS resolution via Route 53"] --> public["Public subnets<br/>ALB origin; inbound restricted to CloudFront"]
  public --> app["Amazon EKS private subnets<br/>ALB routes to App/Worker Pods on EC2 managed node groups"]
  app --> broker["Amazon MQ for RabbitMQ<br/>private multi-AZ; durable/quorum queues"]
  app --> data["Private data subnets<br/>SQL Server EC2 Always On AG + ElastiCache Redis OSS"]
  app --> nat["NAT egress for approved outbound calls"]
  nat --> providers["VNPay / MoMo / OAuth and approved external egress"]
  app --> s3["Amazon S3 via VPC endpoint<br/>private bucket + KMS"]
  app --> cw["CloudWatch / Secrets Manager / ECR endpoints"]
  data --> backup["SQL Agent -> encrypted EBS staging -> S3 KMS/Versioning/Object Lock"]
  backup --> dr["Async AG DR + S3 CRR to isolated backup account/Region; AWS Backup VSS copy"]
  note["Security groups permit broker traffic only from app/worker subnets and database/cache traffic only from approved application subnets. Secondary SQL topology follows D-026; licensing/sizing remain a DBA/Finance go-live gate."]
  data -.-> note
```
<!-- END MERMAID DIAGRAM DEPLOY-VPC -->


---

### AWS shared services and security
- IAM service roles cho CodePipeline/CodeBuild và EKS Pod Identity cho workload; GitHub kết nối qua AWS CodeConnections. Không cấp access key dài hạn cho pipeline hoặc lưu trong GitHub. CodeBuild chỉ được quyền build/push đúng ECR repositories; CodePipeline artifact bucket dùng S3/KMS với retention đã duyệt.
- Secrets Manager giữ DB/payment/OAuth secrets; KMS mã hóa S3/EBS/backup; rotation và least privilege.
- CloudTrail, AWS Config, GuardDuty và Security Hub cho audit/config/threat detection.
- VPC endpoints cho S3, ECR, CloudWatch, Secrets Manager và Systems Manager; NAT per AZ chỉ cho egress đối tác.
- SQL Agent tạo native FULL/DIFF/LOG backup trên EBS staging mã hóa; AWS CLI/SDK uploader dùng IAM instance role chuyển file đã hoàn tất lên S3. S3 bật Versioning, SSE-KMS, Object Lock và CRR sang tài khoản/Region backup; theo dõi replication status/lag. AWS Backup Windows VSS + Vault Lock + Restore Testing phục hồi EC2/EBS, không thay SQL PITR.
- AWS Budgets, Cost Explorer và AWS Pricing Calculator; ghi Region, SKU, request/egress assumptions và ngày định giá.
## 6. CI/CD Pipeline trên AWS CodePipeline

<!-- BEGIN MERMAID DIAGRAM CONVERT-07_Deployment_And_Operations_Architecture -->
### CI/CD pipeline

```mermaid
flowchart TD
  source["GitHub source repository<br/>AWS CodeConnections"]
  pipeline["AWS CodePipeline V2"]
  quality["AWS CodeBuild<br/>JUnit, coverage, SonarQube, Trivy"]
  build["Build, scan and publish immutable container images"]
  registry["Amazon ECR<br/>image tagged by commit/build"]
  staging["CodePipeline EKS deploy action V2<br/>staging cluster"]
  tests["CodeBuild integration / smoke tests"]
  approve{"Manual Tech Lead approval"}
  prod["CodePipeline EKS deploy action V2<br/>production cluster"]
  health{"Health checks pass?"}
  worker["Drain worker messages before shutdown"]
  switch["Shift traffic; graceful old-pod shutdown"]
  fail["Stop rollout and investigate"]
  source --> pipeline --> quality --> build --> registry --> staging --> tests --> approve --> prod --> health
  health -->|yes| switch --> worker
  health -->|no| fail
```
<!-- END MERMAID DIAGRAM CONVERT-07_Deployment_And_Operations_Architecture -->

**Điều kiện AWS:** GitHub được kết nối qua CodeConnections; EKS deploy action chỉ dùng với CodePipeline V2, nhận Kubernetes manifests và chạy trên CodeBuild-managed compute nên có chi phí CodeBuild riêng. Pipeline artifacts lưu trong S3 bucket mã hóa bằng KMS; quyền truy cập private EKS và IAM service roles phải được kiểm thử.



---

## 7. Cấu Hình Biến Môi Trường

| Biến | Mô tả | Ví dụ |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Profile | `prod`, `staging`, `dev` |
| `DB_HOST` | SQL Server host | `sqlserver-primary.internal` |
| `DB_USERNAME` / `DB_PASSWORD` | SQL Server credentials | *(secret)* |
| `REDIS_HOST` / `REDIS_PASSWORD` | Redis cluster | *(secret)* |
| `RABBITMQ_HOST` | RabbitMQ host | `rabbitmq.internal` |
| `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` | RabbitMQ credentials | *(secret)* |
| `JWT_PRIVATE_KEY_PATH` | RS256 private key | `/secrets/jwt-private.pem` |
| `VNPAY_TMN_CODE` / `VNPAY_HASH_SECRET` | VNPay credentials | *(secret)* |
| `MOMO_PARTNER_CODE` / `MOMO_SECRET_KEY` | MoMo credentials | *(secret)* |
      AWS_SES_CONFIGURATION_SET: configured-in-secrets-manager
      SNS_PLATFORM_APPLICATION_ARN: AWS-resource-arn
| `AWS_S3_BUCKET` / `AWS_REGION` | Object storage | `ticket-assets`, `ap-southeast-1` |

---

## 8. Flash-Sale Runbook (Trước Giờ Mở Bán)

| Mốc | Đội phụ trách | Hành động bắt buộc | Mục tiêu |
|---|---|---|---|
| **T - 2 Giờ** | Lead Backend + DevOps | Kiểm tra sức khỏe App, Worker, SQL Server, Redis, RabbitMQ<br>Xác minh RabbitMQ Queue rỗng (không còn message tồn đọng)<br>Kiểm tra hạn mức kết nối VNPay/MoMo | Hạ tầng sạch, sẵn sàng |
| **T - 60 Phút** | Lead Backend | Feature Freeze — ngừng deploy mới<br>Scale-out: tăng App instances lên tối đa<br>Tăng Worker instances (xử lý email nhanh hơn)<br>Pre-load Concert & Seat Map vào Redis Cache | Tránh cold-start khi surge |
| **T - 15 Phút** | Hệ thống tự động | Kích hoạt Virtual Queue (Holding Room)<br>Chuyển hướng tất cả traffic vào queue endpoint<br>Bật Minimal Logging mode | Hấp thụ traffic burst |
| **T = 0** | Hệ thống tự động | Fisher-Yates Shuffle số thứ tự hàng chờ<br>Mở van 5.000 CCU vào khu vực chọn ghế<br>Dev team theo dõi CloudWatch dashboards/alarms: P99, error rate, queue depth | Bán vé chính thức |
| **T + 30 Phút** | PO + Lead Dev | Đóng hàng chờ (vé hết / traffic hạ)<br>Scale-in instances về mức bình thường<br>Đối soát: số vé bán = số tiền nhận = số tickets issued | Hoàn tất đợt bán |

---

## 9. AWS Giám Sát & Quan Sát (Observability)

| Công cụ | Vai trò |
|---|---|
| **Spring Boot Actuator** | `/actuator/health`, `/actuator/metrics` — health check & JVM metrics |
| **Amazon CloudWatch / Container Insights** | Dashboard/alarms latency, error rate, EKS/EC2, JVM và queue metrics |
| **RabbitMQ Management UI** | Monitor Queue depth, Consumer rate, DLQ messages |
| **CloudWatch Logs** | Structured JSON stdout/stderr; retention 30 ngày, PII filtered |
| **CloudWatch Application Signals / AWS X-Ray** | Traces, dependencies, latency/error; SNS alerts |
