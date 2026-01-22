
# 🏗️ Architecture Overview

The application delivery layer follows a **secure, scalable three-tier architecture** optimized for high availability and global performance.

## 🔄 Request Flow

```text
Internet
   ↓ HTTPS + DDoS Protection
CloudFront
   ↓ HTTP
Application Load Balancer (ALB)
   ↓
ECS Fargate (Private Subnets)
   ↓
Backend Services
```

## 🧩 Core Components

* **CloudFront**

  * Global CDN
  * DDoS protection (AWS Shield Standard)
  * SSL/TLS termination
* **Application Load Balancer**

  * Multi-AZ traffic distribution
  * Health checks
* **ECS Fargate**

  * Serverless container orchestration
  * Private networking (no public IPs)
* **Security**

  * End-to-end encryption
  * Least-privilege IAM roles
  * Network isolation via private subnets

---

# 🐳 ECS Fargate Configuration

## 📦 Service Specifications

| Attribute            | Value                        |
| -------------------- | ---------------------------- |
| **Cluster Name**     | `fintech-cluster`            |
| **Service Name**     | `fintech-api-service`        |
| **Desired Tasks**    | 3 (distributed across 2 AZs) |
| **CPU / Memory**     | 512 CPU units / 1024 MB      |
| **Network Mode**     | `awsvpc`                     |
| **Platform Version** | Latest                       |

---

## 📄 Task Definition Details

### 🐋 Container Configuration

| Setting            | Value                                                             |
| ------------------ | ----------------------------------------------------------------- |
| **Image**          | `296674251987.dkr.ecr.us-east-1.amazonaws.com/fintech-api:latest` |
| **Container Port** | 8080                                                              |

### 🌱 Environment Variables

```env
ENVIRONMENT=production
REGION=us-east-1
REDIS_HOST=master.fintech-redis-cluster.qnqmuf.use1.cache.amazonaws.com
REDIS_PORT=6379
```

### 🔐 Secrets Injection (AWS Secrets Manager)

| Secret          | ARN                                                                                      |
| --------------- | ---------------------------------------------------------------------------------------- |
| **DB_PASSWORD** | `arn:aws:secretsmanager:us-east-1:296674251987:secret:fintech/db-credentials:password::` |
| **API_KEY**     | `arn:aws:secretsmanager:us-east-1:296674251987:secret:fintech/api-keys`                  |

---

## 🧑‍💼 IAM Roles

### 🔧 ECS Task Execution Role (`ecsTaskExecutionRole`)

**Purpose:** Used during task startup

**Permissions:**

* Pull images from Amazon ECR
* Write logs to CloudWatch Logs
* Read secrets from AWS Secrets Manager

---

### 🚀 ECS Task Role (`ecsTaskRole`)

**Purpose:** Used at runtime by the application

**Permissions:**

* Full access to `fintech/*` secrets
* DynamoDB CRUD on `fintech-sessions`
* KMS decrypt using CMK
  `d9c8447f-91df-4df9-a012-6a5b6eab3cd0`

---

## 🌐 Networking Configuration

| Setting            | Value                                                  |
| ------------------ | ------------------------------------------------------ |
| **Subnets**        | Private A & B                                          |
| **Subnet IDs**     | `subnet-01808abced051ff76`, `subnet-08f5e862854552155` |
| **Security Group** | `sg-00f1816433ac5e19b`                                 |
| **Public IP**      | Disabled                                               |
| **Logs**           | `/ecs/fintech-api`                                     |

---

## 🔧 Commands Used

```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name fintech-cluster

# Register task definition
aws ecs register-task-definition \
  --family fintech-api \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 512 \
  --memory 1024
```
<img width="1499" height="653" alt="image" src="https://github.com/user-attachments/assets/e177c2c6-c384-4092-92f9-db3845d52b9b" />

<img width="1501" height="419" alt="image" src="https://github.com/user-attachments/assets/f2efb857-425b-49ba-a8c9-08cf093d7fd9" />

<img width="1501" height="498" alt="image" src="https://github.com/user-attachments/assets/228c3417-6c8f-4834-a12c-f0241a427041" />

---

# ⚖️ Application Load Balancer (ALB)

## 🧾 ALB Specifications

| Attribute    | Value                                               |
| ------------ | --------------------------------------------------- |
| **Name**     | `fintech-alb`                                       |
| **Scheme**   | Internal                                            |
| **Type**     | Application                                         |
| **IP Type**  | IPv4                                                |
| **DNS Name** | `fintech-alb-692182147.us-east-1.elb.amazonaws.com` |

<img width="1574" height="271" alt="image" src="https://github.com/user-attachments/assets/998d7af5-b097-4c1d-aa86-40efcb4a153e" />



<img width="1579" height="701" alt="image" src="https://github.com/user-attachments/assets/671622ac-a6f1-4cd9-9e29-e349063b7a23" />

---

## 🎧 Listener Configuration

| Setting            | Value                       |
| ------------------ | --------------------------- |
| **Protocol**       | HTTP                        |
| **Port**           | 80                          |
| **Default Action** | Forward to `fintech-api-tg` |


<img width="1382" height="347" alt="image" src="https://github.com/user-attachments/assets/e297f3ee-5099-4eb7-825c-01624fe8b4f7" />

---

## 🎯 Target Group Configuration

| Setting                 | Value            |
| ----------------------- | ---------------- |
| **Name**                | `fintech-api-tg` |
| **Protocol**            | HTTP             |
| **Port**                | 8080             |
| **Target Type**         | `ip`             |
| **Health Check Path**   | `/health`        |
| **Interval**            | 30s              |
| **Timeout**             | 5s               |
| **Healthy / Unhealthy** | 2 / 2            |


<img width="1588" height="694" alt="image" src="https://github.com/user-attachments/assets/b94504aa-026d-487d-ba6c-31adfd30e33d" />

---

## 🔐 Security Configuration

* **Security Group:** `sg-08990648c310029a9`
* **Inbound**

  * HTTP (80) from CloudFront IP ranges
  * HTTPS (443) from CloudFront IP ranges
* **Outbound**

  * All traffic allowed

<img width="1545" height="254" alt="image" src="https://github.com/user-attachments/assets/611a912f-7808-4737-b190-91d76d496c54" />

---

## 🌍 Multi-AZ Configuration

| AZ         | Public Subnet              |
| ---------- | -------------------------- |
| us-east-1a | `subnet-01a0ea892bddbe272` |
| us-east-1b | `subnet-01e465ff29dc73d6e` |

* **Cross-Zone Load Balancing:** Enabled
* **Deletion Protection:** Disabled (dev environment)

---

## 🔧 Commands Used

```bash
aws elbv2 create-target-group \
  --name fintech-api-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-05ae083309505dd2d \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2
```

---

# ☁️ CloudFront Integration

## 🌐 Distribution Configuration

| Attribute           | Value                                               |
| ------------------- | --------------------------------------------------- |
| **Domain**          | `d17sqls800e6wq.cloudfront.net`                     |
| **Origin**          | `fintech-alb-692182147.us-east-1.elb.amazonaws.com` |
| **Origin Protocol** | HTTP only                                           |
| **Viewer Protocol** | Redirect HTTP → HTTPS                               |
| **Price Class**     | All Edge Locations                                  |
| **IPv6**            | Enabled                                             |

---

## ⚡ Cache Behavior

| Setting             | Value                                        |
| ------------------- | -------------------------------------------- |
| **Allowed Methods** | GET, HEAD, POST, PUT, PATCH, OPTIONS, DELETE |
| **Cached Methods**  | GET, HEAD                                    |
| **Query Strings**   | Forwarded                                    |
| **Cookies**         | None                                         |
| **TTL**             | 0 seconds (API responses not cached)         |

---

## 🔒 Security Features

* Free SSL/TLS certificates
* AWS Shield Standard (DDoS protection)
* HTTP/2 enabled
* ALB access restricted to CloudFront only

---

## 🚧 CloudFront IP Restriction Strategy

* **Total IP ranges:** 80+ global
* **Added:** 58 ranges (hit 60-rule SG limit)
* **Coverage:** Majority of edge locations
* **Production Alternative:**
  **Custom Origin Headers** instead of IP-based rules

---

## 🔧 Commands Used

```bash
CLOUDFRONT_IPS=$(curl -s https://ip-ranges.amazonaws.com/ip-ranges.json \
  | jq -r '.prefixes[] | select(.service=="CLOUDFRONT") | select(.region=="GLOBAL") | .ip_prefix')

for ip in $CLOUDFRONT_IPS; do
  aws ec2 authorize-security-group-ingress \
    --group-id sg-08990648c310029a9 \
    --protocol tcp \
    --port 80 \
    --cidr $ip
done
```

<img width="1523" height="596" alt="image" src="https://github.com/user-attachments/assets/8686e735-263f-4ee8-80d3-a88b8fb0376e" />


<img width="735" height="258" alt="image" src="https://github.com/user-attachments/assets/349debcc-a157-4c46-8bcf-7304cf8ad044" />

---

# 🔒 Security Architecture

## 🌐 Network Security

* Private ALB (no direct internet access)
* CloudFront-only origin access
* ECS tasks in private subnets
* Least-privilege security group rules

---

## 🛡️ Application Security

* Secrets injected at runtime
* KMS-encrypted secrets
* SSL/TLS enforced everywhere:

  * CloudFront → Users
  * Redis (SSL enabled)
  * Amazon RDS PostgreSQL (SSL enforced)

---

## 🧑‍💼 IAM Security

* Clear separation of duties
* Fine-grained permissions
* No wildcard access
* No over-permissioning

---

# 📦 Amazon Elastic Container Registry (ECR)

## 🧾 Repository Configuration

| Attribute              | Value                     |
| ---------------------- | ------------------------- |
| **Repository Name**    | `fintech-api`             |
| **Region**             | us-east-1                 |
| **Image Tag Strategy** | `latest` + versioned tags |
| **Scan on Push**       | Enabled                   |
| **Encryption**         | AES-256 (AWS managed)     |

---

## 🐋 Container Image Usage

The ECS Fargate service pulls container images directly from Amazon ECR during task startup.

```text
ECS Task Execution Role
   ↓
Authenticate to ECR
   ↓
Pull Container Image
   ↓
Start Application Container
```

### Image Reference

```text
296674251987.dkr.ecr.us-east-1.amazonaws.com/fintech-api:latest
```

---

## 🔐 Security & Access Control

### 🔑 IAM Integration

Access to ECR is controlled using IAM roles:

#### ECS Task Execution Role Permissions

* `ecr:GetAuthorizationToken`
* `ecr:BatchCheckLayerAvailability`
* `ecr:GetDownloadUrlForLayer`
* `ecr:BatchGetImage`

This ensures **only ECS tasks** can pull images.

---

### 🌐 Private Connectivity

To avoid public internet exposure, ECR access is routed through **VPC interface endpoints**:

* `com.amazonaws.us-east-1.ecr.api`
* `com.amazonaws.us-east-1.ecr.dkr`

**Benefits:**

* No NAT Gateway dependency
* Reduced attack surface
* Lower data transfer costs

---

## 🔄 Image Lifecycle Strategy

| Stage  | Description                               |
| ------ | ----------------------------------------- |
| Build  | Image built locally or in CI pipeline     |
| Push   | Image pushed to private ECR repository    |
| Deploy | ECS pulls image during task startup       |
| Update | New image tag triggers rolling deployment |

---

## 🔧 Commands Used

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name fintech-api \
  --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login \
    --username AWS \
    --password-stdin 296674251987.dkr.ecr.us-east-1.amazonaws.com

# Build Docker image
docker build -t fintech-api .

# Tag image
docker tag fintech-api:latest \
  296674251987.dkr.ecr.us-east-1.amazonaws.com/fintech-api:latest

# Push image to ECR
docker push \
  296674251987.dkr.ecr.us-east-1.amazonaws.com/fintech-api:latest
```

---

## 🧠 Architectural Benefits of Using ECR

* 🔒 **Private Registry** – No public Docker Hub dependency
* ⚡ **Low Latency** – Same-region image pulls
* 🧩 **Native AWS Integration** – IAM, ECS, CloudWatch
* 💰 **Cost Efficient** – No outbound internet traffic
* 📈 **Scalable** – Supports rapid deployment cycles

---

