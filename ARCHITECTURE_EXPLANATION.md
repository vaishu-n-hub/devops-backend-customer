# AWS EKS Architecture - Complete Explanation

## Architecture Overview
This is a highly available, secure, multi-tier AWS architecture for running containerized applications on EKS (Elastic Kubernetes Service) across 3 availability zones in Ohio (us-east-2).

---

## Components Breakdown

### 1. USER ACCESS LAYER
**Users → Internet → Route 53**
- Users access the application from the internet
- Route 53 (DNS service) routes traffic to the appropriate AWS resources
- Entry point for all external traffic

---

### 2. NETWORK LAYER - VPC (Virtual Private Cloud)

**VPC Structure:**
- Region: Ohio (us-east-2)
- 3 Availability Zones (2a, 2b, 2c) for high availability
- 2 subnet types per AZ:
  - **Public Subnets** (3 total: subnet-1, subnet-2, subnet-3)
  - **Private Subnets** (9 total: 3 for Web Tier, 3 for App Tier, 3 for DB Tier)

**Why 3 Availability Zones?**
- High availability: If one AZ fails, others keep running
- Fault tolerance: Distributes workload across physical data centers
- AWS best practice for production workloads

---

### 3. LOAD BALANCING LAYER

**ALB (Application Load Balancer) - Public**
- Sits in public subnets
- Internet-facing
- Receives traffic from Route 53
- Distributes traffic to EKS worker nodes
- Performs health checks
- Handles SSL/TLS termination

**ALB (Application Load Balancer) - Private**
- Sits between tiers (internal communication)
- Routes traffic from Web Tier to App Tier
- Not accessible from internet
- Used for microservices communication

---

### 4. SECURITY LAYER

**WAF (Web Application Firewall)**
- Attached to the public ALB
- Protects against common web exploits:
  - SQL injection
  - Cross-site scripting (XSS)
  - DDoS attacks
- Filters malicious traffic before it reaches your application

**AWS Certificate Manager (ACM)**
- Manages SSL/TLS certificates
- Enables HTTPS for secure communication
- Free SSL certificates
- Auto-renewal

**Private Certificate Authority (PCA)**
- Issues private certificates for internal communication
- Used for service-to-service encryption within VPC
- Not for public-facing services

---

### 5. KUBERNETES LAYER - EKS

**EKS Control Plane**
- Managed by AWS (you don't see these servers)
- Runs Kubernetes API server, scheduler, controller manager
- Highly available across multiple AZs
- Connected via Private Route 53 for internal DNS

**EKS Worker Nodes (EC2 Instances)**
- Distributed across private subnets in all 3 AZs
- Run your containerized applications (pods)
- Organized in 3 tiers:

#### **Web Tier Subnets (Private subnet-1, 4, 7)**
- Frontend applications
- Receives traffic from public ALB
- Examples: React apps, API gateways, web servers

#### **App Tier Subnets (Private subnet-2, 5, 8)**
- Backend application logic
- Microservices
- Business logic processing
- Examples: Spring Boot apps (like your packersmovers app), Node.js services

#### **DB Tier Subnets (Private subnet-3, 6, 9)**
- Database access layer
- Data processing services
- Closest to RDS database

---

### 6. DATABASE LAYER

**RDS (Relational Database Service)**
- MySQL database (as per your application.properties)
- Sits in private subnets (DB Tier)
- Multi-AZ deployment for high availability
- Automatic backups
- Not directly accessible from internet
- Only accessible from App Tier pods

---

### 7. CONTAINER REGISTRY

**ECR (Elastic Container Registry)**
- Stores your Docker images
- Private registry (not public like Docker Hub)
- Integrated with EKS
- Your GitHub Actions workflow pushes images here
- EKS pulls images from here to run pods

---

### 8. NAT GATEWAY - THE KEY COMPONENT

**Location:** Public subnets (one per AZ for high availability)

**Purpose:** Allows resources in PRIVATE subnets to access the INTERNET while remaining private

### WHY NAT GATEWAY IS CRITICAL:

#### **Use Case 1: Pulling Docker Images**
```
EKS Pod (Private Subnet) → NAT Gateway → Internet → ECR/Docker Hub
```
- Your pods need to pull Docker images from ECR
- ECR endpoints are on the internet (even though it's AWS service)
- Without NAT: Pods can't download images = deployment fails

#### **Use Case 2: Software Updates & Package Downloads**
```
EKS Pod (Private Subnet) → NAT Gateway → Internet → Maven Central/npm/apt
```
- Your Spring Boot app might need to download dependencies
- OS updates, security patches
- Without NAT: Can't update or install packages

#### **Use Case 3: External API Calls**
```
Your App (Private Subnet) → NAT Gateway → Internet → External APIs
```
- Your app might call external services (payment gateways, SMS services, etc.)
- AWS SNS (as per your code) might need internet access
- Without NAT: Can't reach external services

#### **Use Case 4: AWS Service Endpoints**
```
EKS Pod → NAT Gateway → Internet → AWS Services (S3, DynamoDB, etc.)
```
- Many AWS services require internet access (unless you use VPC endpoints)
- CloudWatch logs, metrics
- Secrets Manager, Parameter Store

### HOW NAT GATEWAY WORKS:

1. **Outbound Traffic (Allowed):**
   ```
   Private Subnet Pod → NAT Gateway → Internet
   ```
   - Pod initiates connection to internet
   - NAT Gateway translates private IP to public IP
   - Response comes back through NAT Gateway

2. **Inbound Traffic (Blocked):**
   ```
   Internet → NAT Gateway → Private Subnet ❌ BLOCKED
   ```
   - Internet cannot initiate connections to private resources
   - This keeps your pods secure

### NAT GATEWAY vs INTERNET GATEWAY:

| Feature | Internet Gateway | NAT Gateway |
|---------|-----------------|-------------|
| Purpose | Public subnet internet access | Private subnet outbound access |
| Direction | Bidirectional (in & out) | Outbound only |
| Cost | Free | ~$0.045/hour + data transfer |
| Use Case | Public-facing resources | Private resources needing internet |

---

## TRAFFIC FLOW EXAMPLES

### Example 1: User Accessing Your Application
```
User Browser
  ↓
Internet
  ↓
Route 53 (DNS resolution)
  ↓
WAF (Security filtering)
  ↓
Public ALB (Load balancing)
  ↓
EKS Pod in Web Tier (Private subnet-1)
  ↓
Private ALB (Internal routing)
  ↓
EKS Pod in App Tier (Private subnet-2) - Your Spring Boot app
  ↓
RDS MySQL (Private subnet-3)
```

### Example 2: Your App Calling External API
```
Spring Boot Pod (Private subnet-2)
  ↓
NAT Gateway (Public subnet)
  ↓
Internet Gateway
  ↓
External API (e.g., payment gateway)
```

### Example 3: Deploying New Version via GitHub Actions
```
GitHub Actions (Internet)
  ↓
Builds Docker image
  ↓
Pushes to ECR (via internet)
  ↓
Connects to EKS API (Public endpoint)
  ↓
Helm deploys to EKS
  ↓
EKS pulls image from ECR (via NAT Gateway)
  ↓
Pod starts in Private subnet
```

---

## SECURITY ARCHITECTURE

### Defense in Depth (Multiple Security Layers):

1. **Perimeter Security:**
   - WAF blocks malicious requests
   - Route 53 with DDoS protection

2. **Network Security:**
   - Public subnets: Only ALB and NAT Gateway
   - Private subnets: All application workloads
   - Security Groups: Control traffic between resources
   - Network ACLs: Subnet-level firewall

3. **Application Security:**
   - SSL/TLS encryption (ACM certificates)
   - Private communication (PCA certificates)
   - IAM roles for pod authentication (no hardcoded credentials)

4. **Data Security:**
   - RDS in private subnets (no internet access)
   - Encryption at rest and in transit
   - Automated backups

---

## HIGH AVAILABILITY DESIGN

### How This Architecture Handles Failures:

**Scenario 1: One AZ Goes Down**
- Traffic automatically routes to healthy AZs (2b and 2c)
- ALB health checks detect failures
- RDS fails over to standby in another AZ
- Zero downtime for users

**Scenario 2: Pod Crashes**
- Kubernetes automatically restarts the pod
- ALB stops sending traffic to unhealthy pod
- Other pods handle the load
- Self-healing architecture

**Scenario 3: NAT Gateway Failure**
- Each AZ has its own NAT Gateway
- Failure only affects that AZ
- Other AZs continue functioning
- Route tables ensure proper routing

---

## COST OPTIMIZATION CONSIDERATIONS

### Expensive Components:
1. **NAT Gateway:** ~$32/month per gateway × 3 AZs = ~$96/month
   - Alternative: Single NAT Gateway (less HA)
   - Alternative: VPC Endpoints for AWS services (no NAT needed)

2. **ALB:** ~$16/month per ALB × 2 = ~$32/month

3. **EKS Control Plane:** ~$73/month

4. **EC2 Worker Nodes:** Depends on instance types

### Cost Saving Tips:
- Use VPC Endpoints for S3, ECR, CloudWatch (avoid NAT charges)
- Use single NAT Gateway for dev/test environments
- Use Fargate instead of EC2 for variable workloads
- Enable cluster autoscaler to scale down during low traffic

---

## INTERVIEW TALKING POINTS

### When Asked About NAT Gateway:
"NAT Gateway is essential in our architecture because our EKS worker nodes run in private subnets for security. However, they still need outbound internet access to pull Docker images from ECR, download software updates, and call external APIs. NAT Gateway provides this one-way internet access while keeping our infrastructure secure from inbound internet threats. We deploy one NAT Gateway per availability zone for high availability."

### When Asked About Multi-AZ Design:
"We use 3 availability zones to ensure high availability and fault tolerance. If one AZ experiences an outage, our application continues running in the other two zones. The ALB automatically distributes traffic only to healthy instances, and our RDS database can failover to a standby in another AZ. This design provides 99.99% availability."

### When Asked About Security:
"We implement defense in depth with multiple security layers: WAF at the edge for application-level protection, private subnets for all workloads, security groups for instance-level firewalls, IAM roles for authentication, and encryption in transit using ACM certificates. Our database is completely isolated in private subnets with no internet access."

### When Asked About EKS vs EC2:
"EKS provides container orchestration with automatic scaling, self-healing, and rolling updates. Kubernetes manages pod lifecycle, service discovery, and load balancing. Compared to traditional EC2, we get better resource utilization, faster deployments, and easier management of microservices. The managed control plane reduces operational overhead."

---

## COMMON INTERVIEW QUESTIONS & ANSWERS

**Q: Why not put everything in public subnets?**
A: Security. Private subnets prevent direct internet access to our applications and databases. Only the load balancer needs to be public-facing. This reduces attack surface significantly.

**Q: Can you remove NAT Gateway to save costs?**
A: Not without consequences. Pods wouldn't be able to pull images, download updates, or call external services. Alternative: Use VPC Endpoints for AWS services, but you'd still need NAT for non-AWS internet access.

**Q: Why use ALB instead of NLB?**
A: ALB operates at Layer 7 (HTTP/HTTPS), providing advanced routing based on URL paths, host headers, and HTTP methods. Perfect for microservices. NLB is Layer 4 (TCP/UDP), better for extreme performance needs or non-HTTP protocols.

**Q: How does pod-to-pod communication work?**
A: Kubernetes uses a flat network model. Pods can communicate directly using their internal IPs via the CNI plugin (AWS VPC CNI). No NAT needed for internal communication.

**Q: What happens if NAT Gateway fails?**
A: Only that AZ's private subnets lose internet access. Other AZs continue working. Kubernetes would reschedule pods to healthy AZs. For critical workloads, we have NAT Gateway per AZ.

**Q: How do you handle database credentials securely?**
A: Best practice: Use AWS Secrets Manager or Systems Manager Parameter Store. Pods retrieve credentials at runtime using IAM roles. Never hardcode in application.properties (like currently in your code - mention this as something to improve).

---

## YOUR SPECIFIC IMPLEMENTATION

Based on your code:
- **Application:** Spring Boot (Java 17) packersmovers service
- **Database:** MySQL RDS (endpoint: mysql-rds-instance.cfm6qoakm2ve.us-east-2.rds.amazonaws.com)
- **Container Registry:** ECR
- **Deployment:** GitHub Actions → ECR → EKS via Helm
- **Authentication:** IAM OIDC (no access keys)
- **Monitoring:** Spring Actuator health endpoints

**Your app flow:**
1. User hits ALB
2. ALB routes to your Spring Boot pod (port 8080)
3. Pod connects to RDS MySQL
4. Pod might use AWS SNS for notifications (needs NAT Gateway for this)

---

## IMPROVEMENTS TO SUGGEST IN INTERVIEW

1. **Secrets Management:** Move DB credentials from application.properties to AWS Secrets Manager
2. **Monitoring:** Add CloudWatch Container Insights, Prometheus, Grafana
3. **Logging:** Centralized logging with CloudWatch Logs or ELK stack
4. **Auto-scaling:** Implement HPA (Horizontal Pod Autoscaler) and Cluster Autoscaler
5. **Cost Optimization:** Use VPC Endpoints for ECR, S3, CloudWatch
6. **Disaster Recovery:** Multi-region setup with Route 53 failover
7. **Security:** Implement Pod Security Policies, Network Policies
8. **CI/CD:** Add automated testing, security scanning in pipeline

---

## QUICK REFERENCE - KEY TERMS

- **VPC:** Virtual Private Cloud - Your isolated network in AWS
- **Subnet:** Subdivision of VPC, can be public or private
- **AZ:** Availability Zone - Isolated data center within a region
- **NAT:** Network Address Translation - Allows private → public communication
- **IGW:** Internet Gateway - Allows public subnet internet access
- **ALB:** Application Load Balancer - Layer 7 load balancer
- **EKS:** Elastic Kubernetes Service - Managed Kubernetes
- **ECR:** Elastic Container Registry - Docker image storage
- **RDS:** Relational Database Service - Managed database
- **WAF:** Web Application Firewall - Application-level security
- **ACM:** AWS Certificate Manager - SSL/TLS certificates
- **Route 53:** AWS DNS service

---

Good luck with your interview! Focus on understanding the "why" behind each component, not just the "what."
