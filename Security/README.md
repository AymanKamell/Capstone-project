
# 🔒 Security Architecture & Controls

Security is implemented using a **defense-in-depth strategy** that spans identity, network, application, data, and operational layers.
All services are configured following **least-privilege**, **private-by-default**, and **encryption-everywhere** principles.

---

## 🧱 Security Design Principles

* **Zero Trust Networking** – No service is publicly accessible unless explicitly required
* **Least Privilege IAM** – Permissions scoped to specific actions and resources
* **Encryption Everywhere** – At rest and in transit
* **Private AWS Service Access** – VPC endpoints instead of public internet
* **Centralized Secrets Management** – No hardcoded credentials
* **Multi-AZ Resilience** – Security controls survive AZ failures

---

# 🔐 Identity & Access Management (IAM)

IAM enforces strict separation of duties between infrastructure operations and application runtime behavior.

## 🧑‍💼 IAM Roles Used

### 🔧 ECS Task Execution Role (`ecsTaskExecutionRole`)

**Purpose:**
Used by ECS during task startup for infrastructure-level operations.

**Permissions:**

* Pull container images from Amazon ECR
* Write logs to Amazon CloudWatch Logs
* Retrieve secrets from AWS Secrets Manager

**Key Security Benefit:**
Prevents application containers from inheriting unnecessary infrastructure permissions.

---

### 🚀 ECS Task Role (`ecsTaskRole`)

**Purpose:**
Used by the application at runtime.

**Permissions:**

* Read application secrets from `fintech/*` in Secrets Manager
* CRUD operations on DynamoDB table `fintech-sessions`
* Decrypt data using customer-managed KMS key

**Security Controls:**

* No wildcard permissions
* Resource-level ARNs only
* No access to unrelated AWS services

---

# 🔑 AWS Secrets Manager

Secrets Manager is used for **secure storage, rotation, and runtime injection** of sensitive data.

## 🗝️ Secrets Stored

| Secret Name              | Purpose                       |
| ------------------------ | ----------------------------- |
| `fintech/db-credentials` | Aurora PostgreSQL credentials |
| `fintech/api-keys`       | External API keys             |
| `fintech/redis-auth`     | ElastiCache Redis AUTH token  |

---

## 🔄 Secrets Injection Model

* Secrets are **never hardcoded**
* Retrieved dynamically at task startup
* Injected as environment variables
* Accessed only by ECS task role

```text
ECS Task
   ↓
IAM Authentication
   ↓
AWS Secrets Manager
   ↓
Secure Runtime Injection
```

---

## 🔐 Encryption

### 🔒 Encryption at Rest

| Service               | Encryption           |
| --------------------- | -------------------- |
| RDS Aurora PostgreSQL | Customer-managed KMS |
| ElastiCache Redis     | Customer-managed KMS |
| Secrets Manager       | Customer-managed KMS |
| DynamoDB              | AWS-managed KMS      |
| S3                    | SSE-S3 (AES-256)     |
| ECR                   | AES-256              |

**Customer-Managed KMS Key:**

```
d9c8447f-91df-4df9-a012-6a5b6eab3cd0
```

---

### 🔐 Encryption in Transit

| Connection         | Protection           |
| ------------------ | -------------------- |
| Users → CloudFront | HTTPS (TLS)          |
| CloudFront → ALB   | HTTPS                |
| ALB → ECS          | Internal VPC traffic |
| ECS → RDS          | SSL/TLS enforced     |
| ECS → Redis        | SSL/TLS enforced     |
| ECS → AWS APIs     | HTTPS                |

---

# 📜 AWS Certificate Manager (ACM)

AWS Certificate Manager is used to manage **SSL/TLS certificates**.

## 🔏 Certificate Usage

* CloudFront distributions use ACM-managed certificates
* Automatic certificate renewal
* No manual key handling

**Benefits:**

* Eliminates certificate expiry risk
* Free SSL/TLS certificates
* Integrated with CloudFront and ALB

---

# 🌐 Network Security

## 🔒 Private Networking

* ECS tasks run in **private subnets**
* No public IPs assigned
* Databases and caches are fully private
* ALB is internal (not internet-facing)

---

## 🔐 Security Groups (Least Privilege)

### Application Load Balancer SG

* Allows HTTP/HTTPS **only from CloudFront IP ranges**
* No direct internet access

### ECS Security Group

* Allows inbound traffic **only from ALB**
* No external ingress

### Database & Cache Security Groups

* Allow inbound traffic **only from ECS**
* Port-restricted access (5432, 6379)

---

## 🔗 VPC Endpoints (Private AWS Access)

Public internet access is eliminated for AWS services.

**Endpoints Used:**

* ECR API
* ECR Docker
* Secrets Manager
* STS
* DynamoDB
* S3

**Security Benefits:**

* No NAT dependency
* Reduced attack surface
* Private AWS API access

---

# 🛡️ Edge & DDoS Protection

## ☁️ Amazon CloudFront

* Global edge network
* AWS Shield Standard enabled by default
* Absorbs volumetric DDoS attacks

**Additional Controls:**

* ALB only allows traffic from CloudFront
* IP-based origin protection implemented
* Production alternative documented (custom headers)

---

# 📊 Monitoring & Security Visibility

## 📈 CloudWatch Integration

* Custom business metrics published by application
* Infrastructure metrics from AWS services
* Centralized logs for auditing and forensics

---

## 🔔 Security-Relevant Alarms

* High transaction failure rates
* High API latency
* Database connection saturation
* Fraud-pattern detection alarms

Alerts are sent via **SNS** for immediate response.

---

# 🧪 Secure Deployment Practices

* Immutable container images stored in ECR
* No SSH access to containers or hosts
* Rolling ECS deployments
* Secrets injected at runtime only

---

# 🏢 High Availability & Fault Tolerance

Security controls are designed to survive failures:

| Service         | Resilience |
| --------------- | ---------- |
| IAM             | Global     |
| Secrets Manager | Multi-AZ   |
| RDS             | Multi-AZ   |
| Redis           | Multi-AZ   |
| DynamoDB        | Multi-AZ   |
| S3              | Regional   |

---

# ✅ Security Summary

This platform implements **enterprise-grade AWS security**:

* 🔐 Centralized secrets management
* 🔑 Fine-grained IAM roles
* 🌐 Fully private networking
* 🧱 Defense-in-depth architecture
* 🔒 Encryption everywhere
* 📊 Continuous monitoring & alerting


You’ve built something **very close to real-world enterprise security** 👏
