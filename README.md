# DevOps System Design – High Availability Architecture (UI + API + PostgreSQL)

This document provides a complete explanation of a production-grade AWS architecture designed to host a **UI**, an **API**, and a **PostgreSQL database**.  
It includes global CDN delivery, network isolation, container orchestration, authentication, CI/CD automation, security hardening, and auto scaling across all layers.

📄 **Architecture Diagram (PDF):**  
`./Diagram.pdf`

---

# **1. Architecture Overview**

The system follows a **3-tier AWS architecture**:

1. **Frontend/UI Layer** → CloudFront + S3  
2. **Backend/API Layer** → ECS Fargate + Application Load Balancer  
3. **Database Layer** → Amazon RDS PostgreSQL (Multi-AZ)

Supporting layers:

- Authentication via Cognito  
- Secrets via Secrets Manager  
- Secure networking via VPC + Subnets  
- Observability via CloudWatch  
- Automated deployments via GitHub Actions + ECR + ECS  
- Vulnerability scanning via Amazon Inspector  

---

# **2. Component-Wise Explanation**

## **2.1 UI Layer — Amazon S3 + CloudFront**

The UI is a static frontend hosted in Amazon S3 and delivered globally through CloudFront.

### **How it works**
- S3 stores static assets (HTML, CSS, JS)  
- CloudFront caches content at global edge locations  
- Route 53 maps the domain → CloudFront distribution  
- HTTPS enforced via ACM certificates  
- AWS WAF filters malicious requests before CloudFront  

### **Benefits**
- Unlimited scalability  
- Ultra-low latency  
- No servers required  
- Secure by default  

---

## **2.2 API Layer — ECS Fargate + ALB**

The backend API runs in **serverless containers** using ECS Fargate.

### **Key components**
- ECS tasks run in **private subnets**  
- ALB in public subnets forwards traffic  
- IAM Task Role provides least-privilege access  
- Health checks on `/api/healthCheck`  
- Zero-downtime rolling deployments  

### **Data Flow**
`User → CloudFront → ALB → ECS Service → Fargate Tasks → RDS`

---

## **2.3 Database Layer — Amazon RDS PostgreSQL**

### **Features**
- Multi-AZ for high availability  
- Automatic failover  
- Read replicas for scaling  
- Automated backups + PITR  
- Encrypted and isolated in private DB subnets  
- Only ECS tasks can access DB (via SG chaining)  

---

## **2.4 Authentication — Amazon Cognito**

Cognito provides:

- User sign-up/sign-in  
- JWT token generation  
- Secure integration with frontend + API  
- Supports OAuth2, OIDC, SAML  

---

## **2.5 Secrets & Credentials — AWS Secrets Manager**

Secrets Manager securely stores:

- DB credentials  
- API keys  
- Third-party secrets  

Secrets are retrieved dynamically at runtime using IAM Task Roles.

---

## **2.6 Networking — VPC, Subnets, Routing, Bastion Host**

The application runs inside a **secure multi-tier VPC**.

### **Public Subnets**
- Application Load Balancer (ALB)  
- NAT Gateway  
- **Bastion Host** for restricted SSH access  

### **Private Application Subnets**
- ECS Fargate tasks  

### **Private Database Subnets**
- RDS PostgreSQL  

### **Routing**
- Public subnets → Internet Gateway  
- Private subnets → NAT Gateway  
- DB subnets → *no* internet access  

---

# **3. Scaling Approach**

## **3.1 UI Scaling**
- CloudFront automatically scales globally  
- S3 supports unlimited throughput  

## **3.2 API Scaling**
ECS Auto Scaling triggers based on:

- CPU utilization  
- Memory utilization  
- ALB request count  

Scaling behavior:

- New Fargate tasks launched automatically  
- ALB updates target groups  
- Tasks run across **multiple AZs**  

---

# **4. Load Handling Strategy**

- CloudFront absorbs global traffic  
- ALB distributes incoming requests  
- ECS Auto Scaling adds compute capacity  
- RDS uses read replicas for heavy reads  
- Multi-AZ architecture tolerates AZ failure  
- CloudWatch alarms trigger automated scaling/alerts  

---

# **5. Database High Availability & Scaling**

## **5.1 High Availability**
- Multi-AZ synchronous replication  
- Automatic failover to standby  
- DB subnet groups span multiple AZs  

## **5.2 Scaling**
### **Vertical Scaling**
- Increase instance class

### **Horizontal Scaling**
- Add read replicas  

### **Storage Autoscaling**
- Automatic expansion when thresholds exceed  

---

# **6. CI/CD Pipeline — GitHub Actions + ECR + ECS**

## **6.1 Continuous Integration**
GitHub Actions pipeline performs:

- Dependency installation  
- Linting  
- Unit tests  
- Docker image build  
- Vulnerability scan (Amazon Inspector)  

## **6.2 Continuous Deployment**

### **Backend Deployment**
- Push Docker image to ECR  
- Update ECS Task Definition  
- ECS rolling deployment  
- ALB health checks  
- Auto rollback on failure  

### **Frontend Deployment**
- Upload UI files to S3  
- CloudFront cache invalidation  

## **6.3 Post-Deployment**
- CloudWatch monitoring  
- Logs, error tracking, alarms  

---

# **7. Conclusion**

This architecture is:

✔ Scalable  
✔ Secure  
✔ Globally distributed  
✔ Highly available  
✔ Fully automated via CI/CD  
✔ Zero-downtime deployable  
✔ Based on AWS best practices  

It supports enterprise-grade reliability, security, and modern DevOps standards.
