# **DevOps System Design – High Availability Architecture (UI + API + PostgreSQL)**

This document provides a complete explanation of a **Production-grade AWS architecture** designed to host a **UI**, an **API**, and a **PostgreSQL database**.

It includes global CDN delivery, secure VPC networking, container orchestration, authentication, CI/CD automation, code quality enforcement, database security, vulnerability scanning, and auto scaling across all layers aligned with AWS Well-Architected Framework principles.

**📄 Architecture Diagram (PDF):**
./Diagram.pdf

---

# **1. Architecture Overview**

The architecture uses a **3-tier, highly available AWS design**:

1. **Frontend/UI Layer** → CloudFront + S3
2. **Backend/API Layer** → ECS Fargate + Application Load Balancer
3. **Database Layer** → Amazon RDS PostgreSQL (Multi-AZ) + RDS Proxy

Supporting Components:

* Authentication via **Amazon Cognito**
* Secret storage via **AWS Secrets Manager**
* Secure networking via **VPC + Multi-AZ Subnets + NEW Regional NAT Gateway**
* Observability via **CloudWatch**
* CI/CD automation via **GitHub Actions**
* Container scanning via **Amazon Inspector**
* Static code analysis via **SonarQube / SonarCloud**
* IAM roles for workload permissions

---

# **2. Component-Wise Explanation**

## **2.1 UI Layer — Amazon S3 + CloudFront**

The UI is hosted in Amazon S3 and delivered globally through CloudFront.

### **How it works**

* S3 stores React Components
* CloudFront caches content globally
* Route 53 routes domain → CloudFront
* ACM certificate enables HTTPS
* WAF filters malicious traffic

### **Benefits**

* Serverless & infinitely scalable
* Low latency
* High durability
* Built-in DDoS protection

---

## **2.2 API Layer — ECS Fargate + ALB**

The backend API is deployed in containers on ECS Fargate.

### **Key Components**

* Fargate tasks run in **private subnets**
* ALB in public subnets forwards traffic to ECS
* TLS termination via ACM certificate
* IAM Task Roles for least-privilege access
* Health checks from ALB
* Rolling deployments with zero downtime

### **API Traffic Flow**

```
User → CloudFront → ALB → ECS Fargate → RDS Proxy → PostgreSQL
```

---

## **2.3 Database Layer — RDS PostgreSQL (Multi-AZ)**

### **Key Features**

* Multi-AZ for high availability
* Automatic failover
* Encrypted at rest and in transit
* Automated backups + PITR
* Optional read replicas
* RDS Proxy for connection pooling & faster failovers

### **Access**

* Only ECS → RDS Proxy → RDS
* Bastion Host for developer access (restricted)

---

## **2.4 Authentication — Amazon Cognito**

Cognito enables:

* User authentication & user pools
* JWT tokens (ID, access, refresh)
* OAuth2/OIDC/SAML support
* Secure integration with CloudFront and API

---

## **2.5 Secrets Management — AWS Secrets Manager**

Stores:

* DB credentials
* API keys
* Third-party credentials

Secrets are dynamically retrieved by ECS tasks.

---

## **2.6 Networking — VPC, Subnets, Bastion Host & Regional NAT Gateway**

The architecture runs in a Multi-AZ VPC with three subnet tiers.

### **Public Subnets**

* Application Load Balancer
* **Regional NAT Gateway (NEW AWS Service — Used Here)**
* Bastion Host

### **Private Application Subnets**

* ECS Fargate tasks
* Outbound internet via Regional NAT Gateway

### **Private Database Subnets**

* RDS PostgreSQL + RDS Proxy
* NO internet routes

### **Routing**

* Public → Internet Gateway
* Private → **Regional NAT Gateway**
* DB → Only internal routing

---

# ⭐ **NEW: AWS Regional NAT Gateway (Used in This Architecture)**

AWS recently released the **Regional NAT Gateway** — this architecture uses it.

### **Why it is superior**

* **One NAT per region**, not per AZ
* **Up to 76% cost savings**
* **Region-wide high availability**
* **Automatic multi-AZ failover**
* Simplifies routing tables
* Same bandwidth performance as zonal NAT Gateways

### **Benefits for ECS Fargate**

* Secure outbound access to pull images or APIs
* Lower operational cost
* No multiple NAT gateways needed

---

# **3. Scaling Approach**

## **3.1 UI Scaling**

* CloudFront automatically scales
* S3 supports unlimited throughput

## **3.2 API Scaling**

ECS Service Auto Scaling based on:

* CPU
* Memory
* ALB RequestCount per target

Tasks launch across multiple AZs.

## **3.3 Database Scaling**

* Vertical scaling (bigger instance class)
* Horizontal scaling (read replicas)
* Storage autoscaling

---

# **4. Load Handling Strategy**

* CloudFront absorbs global traffic spikes
* ALB distributes load across ECS tasks
* ECS Auto Scaling adds more Fargate tasks
* RDS Proxy stabilizes connection management
* Multi-AZ failover ensures uptime
* CloudWatch alarms trigger automated responses

---

# **5. Database High Availability & Scaling**

### **High Availability**

* Multi-AZ synchronous replication
* Automatic failover
* RDS Proxy reduces failover disruption

### **Scaling**

* Vertical scaling
* Read replicas
* Storage autoscaling

---

# **6. Critical Database Security — Security Groups + IAM DB Authentication**

### **6.1 Security Groups**

Security Groups enforce strict firewall boundaries.

* Only ECS Task SG → RDS Proxy SG → RDS SG
* Bastion Host SG allowed (restricted)
* Prevents unauthorized access
* Protects DB even if misconfigured
* Prevents lateral movement

### **6.2 IAM DB Authentication**

IAM Auth removes static passwords.

* Short-lived IAM tokens (15 mins)
* No hardcoded DB credentials
* All logins recorded in CloudTrail
* Seamless with RDS Proxy

### **6.3 Defense-In-Depth Model**

| Layer        | Protects               |
| ------------ | ---------------------- |
| **SG**       | Who can *reach* the DB |
| **IAM Auth** | Who can *log in* to DB |

---

# **7. CI/CD Pipeline — GitHub Actions + ECR + ECS + SonarQube/SonarCloud**

The CI/CD pipeline ensures **code quality**, **security**, and **automated deployments**.

---

## **7.1 Continuous Integration (CI)**

### **✔️ Linting & Style Checks**

Ensures code consistency.

### **✔️ Unit Tests**

All test suites are executed:

* Jest (Node.js)
* PyTest (Python)
* JUnit (Java)
* Go Test (Go)

### **✔️ Code Coverage**

* Coverage reports generated
* Minimum coverage threshold enforced (e.g., 80%)
* Coverage uploaded to:

  * GitHub
  * **SonarQube/SonarCloud**

### **✔️ Static Code Analysis — SonarQube / SonarCloud**

Your pipeline integrates Sonar for:

* Security hotspots
* Bug detection
* Code smells
* Cyclomatic complexity
* Coverage & duplication percentage
* OWASP dependency scanning
* Quality Gate enforcement (must pass to deploy)

### **Sonar Integration Steps**

1. Add `SONAR_TOKEN` to GitHub secrets
2. CI job runs
3. SonarCloud/SonarQube dashboard displays:
* Coverage trends
* Hotspots
* Vulnerability reports
* Maintainability score

### **Quality Gate**

Deployment is blocked unless:

* Code coverage > threshold
* No critical vulnerabilities
* No new major code smells

### **✔️ Docker Build**

Images built and pushed to ECR.

### **✔️ Vulnerability Scan**

Amazon Inspector scans image for CVEs.

---

## **7.2 Continuous Deployment (CD)**

### **Backend Deployment**

* Update ECS Task Definition
* Rolling update
* Multi-AZ task scheduling
* Health checks
* Auto rollback

### **Frontend Deployment**

* Build → Upload to S3
* CloudFront cache invalidation

---

## **7.3 Post-Deployment**

* CloudWatch metrics & logs
* Alarm → SNS notification
* ECS task & ALB monitoring
* Error detection & rollback capability

---

# **8. Observability & Monitoring**

* CloudWatch Metrics (CPU, memory, ALB latency)
* CloudWatch Logs (ECS, ALB, API)
* RDS Enhanced Monitoring
* SNS alerts on thresholds
* Inspector CVE reports
* SonarCloud dashboard for code quality metrics

---

# **9. Conclusion**

This architecture is:

✔ Highly available
✔ Secure across networking, identity, and database layers
✔ Cost optimized (Regional NAT Gateway)
✔ Scalable across UI, API, DB tiers
✔ Enforced with **unit tests, coverage gates, and SonarQube analysis**
✔ Fully automated through CI/CD
✔ Globally distributed with CloudFront
✔ Zero-downtime deployable
✔ Production ready & AWS Well-Architected aligned

It delivers enterprise-grade reliability, security, maintainability, and performance.


