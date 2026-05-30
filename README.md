# 3tier-aws-project

### What is 3-Tier Architecture?

A **3-tier architecture** separates an application into three logical and independent layers:

1. **Presentation Tier (Web Tier / Frontend)** — Handles user requests and displays UI.
2. **Application Tier (Business Logic / Backend)** — Processes requests, applies rules, and interacts with the database.
3. **Data Tier (Database)** — Stores and manages persistent data.

### Why this design matters ?

1. Security -  Isolate layers (web in public, app & DB in private).
2. Maintainability - Changes in one tier don’t break others.
3. Scale each tier independently.
4. Multi-AZ deployment survives AZ failures.

## Design the architecture for this requirement

![image.png](attachment:ee6c1bf8-46ee-4b14-bf4d-5909eb2c8a3f:image.png)

## Deep Technical Architecture Overview

Here’s how it looks in AWS (standard production-style design):

### **1. Networking Foundation (VPC Layer)**

- **VPC** (e.g., CIDR: 10.0.0.0/16)
- **Availability Zones**: Minimum 2 AZs (e.g., us-east-1a and us-east-1b) for high availability.
- **Subnets** (6 subnets total recommended):
    - **2 Public Subnets** (Web Tier) — 10.0.1.0/24 & 10.0.2.0/24
    - **2 Private Subnets (App Tier)** — 10.0.3.0/24 & 10.0.4.0/24
    - **2 Private/Isolated Subnets (DB Tier)** — 10.0.5.0/24 & 10.0.6.0/24

**Routing**:

- **Public Subnets**: Route Table with **Internet Gateway (IGW)** → Allows inbound internet traffic.
- **Private Subnets (App)**: Route Table with **NAT Gateway** (in public subnet) → Allows outbound internet (for updates, patches) but **no inbound** from internet.
- **DB Subnets**: No direct internet access.

**Key Services**:

- Internet Gateway (IGW)
- NAT Gateway (one per AZ for HA, or one for cost optimization in learning)
- Route Tables (separate for public & private)

### **2. Presentation Tier (Web Tier)**

- **Internet-facing Application Load Balancer (ALB)** placed in **public subnets**.
- **Target Group** → Points to EC2 instances (or Auto Scaling Group).
- **EC2 Instances / Auto Scaling Group (ASG)** in **public subnets** (running Nginx/Apache + static files or simple frontend).
- **Security Group (Web-SG)**:
    - Inbound: HTTP (80) / HTTPS (443) from 0.0.0.0/0
    - Outbound: Allow traffic to App Tier on specific port (e.g., 8080)

**Flow**: User → ALB → Web EC2 → App Tier

### **3. Application Tier (Business Logic)**

- **Internal ALB** (optional but recommended for better separation) or direct from Web Tier.
- **EC2 Instances / ASG** in **private subnets** (running Node.js, Python Flask/Django, Java, etc.).
- **Security Group (App-SG)**:
    - Inbound: Only from **Web-SG** on application port (e.g., 8080).
    - Outbound: To **DB-SG** on DB port (e.g., 3306 for MySQL).

**Role**: Receives requests from Web Tier, processes logic, queries DB, and returns response.

### **4. Data Tier**

- **Amazon RDS** (MySQL / PostgreSQL / Aurora) in **private subnets**.
- Multi-AZ deployment for automatic failover.
- **Security Group (DB-SG)**:
    - Inbound: Only from **App-SG** on port 3306/5432.
    - No direct internet or web tier access.

**Optional Enhancements**:

- Read Replicas for scaling reads.
- Parameter Groups + Option Groups.

### Complete Request-Response Workflow (Connecting the Dots)

1. User opens browser → DNS resolves to **ALB DNS**.
2. **Internet-facing ALB** (public) receives request → Health checks + routing rules.
3. ALB forwards to **Web Tier EC2** (in public subnet).
4. Web server (Nginx/Apache) serves static content or forwards dynamic requests to **App Tier**.
5. Web EC2 calls **Internal ALB** or directly to **App EC2** (private subnet) over private IP.
6. **App EC2** runs business logic → Connects to **RDS** using private IP + DB security group.
7. RDS returns data → Response flows back the same path.

**All internal communication stays inside the VPC** (secure & low latency).

### Security & High Availability Features

- **Security Groups** act as stateful firewalls at instance level.
- **IAM Roles** attached to EC2 (least privilege).
- **Multi-AZ** + **Auto Scaling** → Survives AZ failure.
- **Bastion Host / Session Manager** for secure SSH access to private instances.
- **CloudWatch + SNS** for monitoring & alerts.

### Security Groups for 3-Tier Architecture

### **1. Internet-Facing ALB Security Group (ALB-Web-SG)**

**Purpose**: Front door of the application.

**Inbound Rules:**

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| HTTP | 80 | 0.0.0.0/0 | Allow all users to access the website |
| HTTPS | 443 | 0.0.0.0/0 | Production standard (recommended) |

**Outbound Rules**: Default (All traffic) → ALB needs to forward traffic to Web EC2 instances.

### **2. Web Tier EC2 Security Group (Web-SG)**

**Purpose**: Runs Nginx/Apache, serves static content or proxies to App tier.

**Inbound Rules:**

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| HTTP | 80 | ALB-Web-SG | Only from Internet-facing ALB |
| HTTPS | 443 | ALB-Web-SG | If using HTTPS on EC2 |
| Custom TCP | 8080 | ALB-Web-SG (optional) | If Web tier also receives dynamic traffic |

**Outbound Rules:**

| Type | Port | Destination |  |
| --- | --- | --- | --- |
| Custom TCP | 8080 | App-SG | Web tier talks to Application tier |
| HTTPS | 443 | 0.0.0.0/0 | For OS updates, package downloads |
| HTTP | 80 | 0.0.0.0/0 | For OS updates |

---

### **3. Internal ALB Security Group (ALB-App-SG)**

**Inbound Rules:**

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| Custom TCP | 8080 | Web-SG | Only Web tier can reach Internal ALB |

**Outbound**: Default All Traffic → to App EC2.

### **4. Application Tier EC2 Security Group (App-SG)**

**Purpose**: Runs business logic (Node.js, Python, Java, etc.)

**Inbound Rules:**

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| Custom TCP | 8080 | ALB-App-SG or Web-SG | Only from Web tier / Internal ALB |

**Outbound Rules:**

| Type | Port | Destination | Reason |
| --- | --- | --- | --- |
| Custom TCP | 3306 | DB-SG | Connect to MySQL |
| Custom TCP | 5432 | DB-SG | If using PostgreSQL |
| HTTPS | 443 | 0.0.0.0/0 | Package updates, API calls to other services |

### **5. Database Tier Security Group (DB-SG)**

**Purpose**: RDS (MySQL/PostgreSQL)

**Inbound Rules:**

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| MySQL | 3306 | App-SG | Only Application tier can connect |


**Outbound Rules**: Usually default (All) or restricted.

### Load Balancer

**1. External Load Balancer (Public-Facing Application Load Balancer)**

- **Role**: This acts as the entry point for all client traffic.
- **Functionality**:
    - Distributes incoming client requests to the web tier EC2 instances.
    - Ensures even distribution of traffic for better performance and reliability.
    - Performs health checks to ensure only healthy instances receive traffic.

**2. Web Tier**

- **Role**: Serves the front-end of the application and redirects API calls.
- **Components**:
    - **Nginx Webservers**: Running on EC2 instances.
    - **React.js Website**: The front-end application served by Nginx.
- **Functionality**:
    - **Serving the Website**: Nginx serves the static files for the React.js application to the clients.
    - **Redirecting API Calls**: Nginx is configured to route API requests to the internal-facing load balancer of the application tier.

**3. Internal Load Balancer (Application Tier Load Balancer)**

- **Role**: Manages traffic between the web tier and the application tier.
- **Functionality**:
    - Receives API requests from the web tier.
    - Distributes these requests to the appropriate EC2 instances in the application tier.
    - Ensures high availability and load balancing within the application tier.

**4. Application Tier**

- **Role**: Handles the application logic and processes API requests.
- **Components**:
    - **Node.js Application**: Running on EC2 instances.
- **Functionality**:
    - **Processing Requests**: The Node.js application receives API requests, performs necessary computations or data manipulations.
    - **Database Interaction**: Interacts with the Aurora MySQL database to fetch or update data.
    - **Returning Responses**: Sends the processed data back to the web tier via the internal load balancer.

**5. Database Tier (Aurora MySQL Multi-AZ Database)**

- **Role**: Provides reliable and scalable data storage.
- **Functionality**:
    - **Data Storage**: Stores all the application data in a structured format.
    - **Multi-AZ Setup**: Ensures high availability and fault tolerance by replicating data across multiple availability zones.
    - **Data Retrieval and Manipulation**: Handles queries and transactions from the application tier to manage the data.

**Additional Components**

**Load Balancing**

- **Purpose**: Distributes incoming traffic evenly across multiple instances to prevent any single instance from becoming a bottleneck.
- **Implementation**:
    - **Web Tier**: The external load balancer distributes traffic to web servers.
    - **Application Tier**: The internal load balancer distributes API requests to application servers.

**Health Checks**

- **Purpose**: Continuously monitors the health of instances to ensure only healthy instances receive traffic.
- **Implementation**:
    - **Web Tier**: Health checks by the external load balancer to ensure web servers are responsive.
    - **Application Tier**: Health checks by the internal load balancer to ensure application servers are operational.

**Auto Scaling Groups**

- **Purpose**: Automatically adjusts the number of running instances based on traffic load to maintain performance and cost efficiency.
- **Implementation**:
    - **Web Tier**: Auto-scaling based on metrics like CPU usage or request count to add or remove web server instances.
    - **Application Tier**: Auto-scaling based on similar metrics to adjust the number of application server instances.

**AWS Certificate Manager (ACM)**

- **Purpose**: Manages SSL/TLS certificates to secure data in transit between clients and your application, ensuring encrypted communication.
- **Implementation**:
    - **Certificate Provisioning**: ACM provides and manages SSL/TLS certificates for your domain `learnaws.co.in`.
    - **Certificate Deployment**: The ACM certificates are associated with the public-facing Application Load Balancer (ALB) to enable HTTPS traffic.
    - **Automatic Renewal**: ACM automatically renews certificates before they expire, ensuring uninterrupted secure connections.

**Amazon Route 53**

- **Purpose**: Manages DNS records and directs user traffic to the appropriate AWS resources, optimizing for performance and reliability.
- **Implementation**:
    - **DNS Management**: Route 53 handles DNS queries for the domain `learnaws.co.in`, translating it into IP addresses for your Application Load Balancer.
    - **Traffic Routing**: Route 53 directs client requests to the public-facing Application Load Balancer based on DNS records.
    - **Health Checks and Failover**: Optionally, Route 53 performs health checks on your endpoints and can automatically reroute traffic to healthy resources if needed.

**Summary**

This architecture ensures high availability, scalability, and reliability by distributing the load, monitoring instance health, and scaling resources dynamically. The web tier serves the front-end and routes API calls, the application tier handles business logic and interacts with the database, and the database tier provides robust data storage and retrieval.
