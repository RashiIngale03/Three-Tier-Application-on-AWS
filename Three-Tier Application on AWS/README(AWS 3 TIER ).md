# Three-Tier Application on AWS

A production-style, highly available three-tier web application deployed on AWS — featuring a React/Vite frontend, a Spring Boot backend, and a MySQL (RDS) database — fronted by an Application Load Balancer, secured with ACM/TLS and AWS WAF, and served under a custom domain via Route 53.

## Architecture Overview

```
                                   Route 53 (raashiingale.space)
                                            │
                                        AWS WAF
                                            │
                              Application Load Balancer (ALB)
                                    /                 \
                          HTTP:80 (→ 443 redirect)   HTTPS:443
                                                       /        \
                                            frontend-tg        backend-tg
                                             (path: /)        (path: /api/*)
                                               │                   │
                                    ┌──────────┴─────────┐  ┌──────┴──────────┐
                                    │   Frontend ASG      │  │  Backend ASG     │
                                    │ (public subnets)    │  │ (private subnets)│
                                    │  Nginx + React/Vite │  │  Spring Boot     │
                                    └─────────────────────┘  └────────┬─────────┘
                                                                      │
                                                              Amazon RDS (MySQL)
                                                            (private subnets)

Bastion Host (public-subnet-1) → SSH access to private backend instances
```

### Network Layout

| Component | CIDR | Type |
|---|---|---|
| VPC (`my-vpc`) | `10.0.0.0/16` | — |
| public-subnet-1 | `10.0.0.0/16` | Public |
| public-subnet-2 | `10.0.1.0/16` | Public |
| private-subnet-3 | `10.0.2.0/16` | Private |
| private-subnet-4 | `10.0.3.0/16` | Private |

- **Public subnets** route out via an **Internet Gateway**.
- **Private subnets** route out via a **NAT Gateway** (for updates/package installs only — not publicly reachable).

## Tech Stack

- **Frontend:** React + Vite, served via Nginx
- **Backend:** Java 17, Spring Boot (Maven build)
- **Database:** Amazon RDS (MySQL)
- **Compute:** EC2 Auto Scaling Groups (frontend & backend), Bastion Host
- **Networking:** VPC, public/private subnets, IGW, NAT Gateway, ALB, Target Groups
- **DNS & TLS:** Route 53, AWS Certificate Manager (ACM)
- **Security:** Security Groups, AWS WAF (IP-based blocking rule)

## Prerequisites

- An AWS account with permissions to create VPC, EC2, RDS, ELB, Route 53, ACM, and WAF resources
- A registered domain (e.g. via GoDaddy) to delegate to Route 53
- The application source code (`EasyCRUD-Updated-df`) containing `backend/` and `frontend/` directories
- Your public IP address (for the WAF allow rule)

---

## Setup Guide

### 1. Networking

1. Create a VPC — `10.0.0.0/16`.
2. Create subnets:
   - `public-subnet-1` — `10.0.0.0/16`
   - `public-subnet-2` — `10.0.1.0/16`
   - `private-subnet-3` — `10.0.2.0/16`
   - `private-subnet-4` — `10.0.3.0/16`
3. Create an **Internet Gateway** and attach it to the VPC.
4. Create a **NAT Gateway** (in a public subnet).
5. Create a **public route table**:
   - Route → Internet Gateway
   - Associate with `public-subnet-1` & `public-subnet-2`
6. Create a **private route table**:
   - Route → NAT Gateway
   - Associate with `private-subnet-3` & `private-subnet-4`

### 2. Launch Templates & Auto Scaling Groups

7. Create a **frontend launch template**:
   - Enable auto-assign public IP
   - Attach a security group scoped to the VPC
8. Create a **backend launch template**:
   - Attach a security group scoped to the VPC
9. Create **backend-ASG**:
   - Template: backend | VPC: `my-vpc` | Subnets: private-subnet-3 & 4
   - Desired: 1 | Min: 1 | Max: 2
10. Create **frontend-ASG**:
    - Template: frontend | VPC: `my-vpc` | Subnets: public-subnet-1 & 2
    - Desired: 1 | Min: 1 | Max: 2

### 3. Bastion Host & Database

11. Create a **bastion-host-server** in `public-subnet-1` (VPC: `my-vpc`) for SSH access into private instances.
12. Create an **RDS MySQL** instance:
    - Username: `admin` | Password: `admin123`
    - Connect it to the backend EC2 instance(s)

### 4. Backend Configuration

Access the backend instance via the bastion host, then:

```bash
apt update
apt install mysql-client -y
mysql -u admin -h <rds-endpoint> -padmin123

# inside MySQL:
create database student_db;
show databases;
```

Install runtime & build the app:

```bash
apt install openjdk-17-jdk -y
apt install maven -y
cd EasyCRUD-Updated-df/backend/src/main/resources/
vim application.properties
# set: username=admin | password=admin123 | database=<rds-endpoint>

mvn clean package
cd target
nohup java -jar student-registration-backend-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
lsof -i :8080
```

### 5. Frontend Configuration

On the frontend instance:

```bash
apt update
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
apt install nginx -y
systemctl start nginx

vim .env
# VITE_API_URL=/api   # request mapping

npm install
npm run build
rm -rf /var/www/html/*
cp -rf dist/* /var/www/html/
systemctl restart nginx
```

### 6. Target Groups & Load Balancer

15. Create **frontend-tg**: VPC `my-vpc` | Path `/` | Target: frontend | Port `80`
16. Create **backend-tg**: VPC `my-vpc` | Path `/api/users` | Target: backend | Port `8080`
17. Create an **Application Load Balancer (ALB)**:
    - AZs: public-subnet-1 & 2
    - Default forward → frontend-tg
18. Add a **listener rule** on `HTTP:80`:
    - Path `/api/*` → forward to backend-tg

### 7. DNS with Route 53

19. Create a **Hosted Zone**: `raashiingale.space` → copy nameservers into your domain registrar (e.g. GoDaddy).
20. Create a **record** in the hosted zone:
    - Alias | Region: `ap-south-1` | Type: Application Load Balancer → select the ALB
    - Routing policy: Simple

21. Verify:
    - `http://raashiingale.space` → frontend
    - `http://raashiingale.space/api/` → backend

### 8. HTTPS with ACM

22. Request an **ACM certificate** for `raashiingale.space` and create the validation record in Route 53.
23. Confirm the certificate validates (Route 53).
24. Attach the certificate to the ALB:
    - Add listener → Protocol `HTTPS` | Port `443` → forward to frontend-tg | Certificate: `raashiingale.space`
25. Add a rule on `HTTPS:443`:
    - Path `/api/*` → forward to backend-tg
26. Redirect `HTTP:80` → `HTTPS:443` (edit listener → redirect to URL, protocol HTTPS, port 443, keep custom host/path/query).

27. Verify:
    - `https://raashiingale.space` → frontend
    - `https://raashiingale.space/api/` → backend

### 9. Web Application Firewall (WAF)

28. Create a WAF setup:
    - **IP Set** — `my-ip-set`, scope: Regional, containing your public IP (e.g. `49.204.165.149/32`)
    - **Web ACL** — `my-web-acl`, category: Other, focus: Web, resource: the ALB, starting with an empty rule set
    - Add a **custom rule**: `my-rule` — Action: **Block** — Statement: match the existing IP set

29. Verify from your own IP → `https://raashiingale.space` returns **403 Forbidden**.
30. Verify from a different network → the site loads normally.

---

## Result

A fully functional, TLS-secured, auto-scaling three-tier application reachable at:

- 🌐 `https://raashiingale.space` — Frontend
- 🔌 `https://raashiingale.space/api/` — Backend API

Protected by AWS WAF IP-based access rules, with all HTTP traffic redirected to HTTPS.

## Notes & Improvements

> ⚠️ The credentials above (`admin` / `admin123`) are placeholders from the build process. For any real deployment, replace them with strong, unique secrets managed via **AWS Secrets Manager** or **SSM Parameter Store**, and avoid committing them to version control.

Possible next steps:
- Move RDS credentials and app config into Secrets Manager / Parameter Store
- Add HTTPS health checks and CloudWatch alarms for the ASGs
- Parameterize the setup with Terraform / CloudFormation for repeatable deployments
- Restrict the backend security group to only allow traffic from the ALB and bastion host
