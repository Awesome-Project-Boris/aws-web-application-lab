# AWS Web Application Lab

This repository contains the sample code and configuration scripts for the **Building a Highly Available, Scalable Web Application** AWS Academy capstone project. Throughout the lab you will:

- Deploy a Node.js/Express web app on EC2
- Use Amazon RDS (MySQL) managed database
- Store credentials securely in AWS Secrets Manager
- Configure networking (VPC, subnets, security groups)
- Implement high availability and scalability (ELB, Auto Scaling)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Repository Structure](#repository-structure)
4. [Setup & Deployment](#setup--deployment)
   - [Phase 2: App Server Provisioning](#phase-2-app-server-provisioning)
   - [Phase 3: Decoupling & Secrets](#phase-3-decoupling--secrets)
5. [Running the App Locally](#running-the-app-locally)
6. [Accessing the Application](#accessing-the-application)
7. [Troubleshooting](#troubleshooting)
8. [Cleanup](#cleanup)

---

## Prerequisites

- AWS account with permissions to:
  - Launch EC2 instances (t3.micro)
  - Create an RDS MySQL database
  - Use AWS Secrets Manager
  - Configure Security Groups, Load Balancer, and Auto Scaling
- AWS CLI v2 installed and configured
- Node.js (v14+) and npm installed locally (for testing)
- Git

---

## Architecture Overview

![Architecture Diagram]([docs/architecture-diagram.png](https://raw.githubusercontent.com/Awesome-Project-Boris/aws-web-application-lab/refs/heads/main/%D7%93%D7%99%D7%90%D7%92%D7%A8%D7%9E%D7%AA%20AWS.png))

1. **VPC** with public and private subnets (minimum two AZs)
2. **RDS** (MySQL) in private subnets
3. **EC2 App Servers** in public subnets, behind an **Application Load Balancer**
4. **Secrets Manager** stores DB credentials
5. **Auto Scaling Group** for the web tier

---

## Repository Structure

```
├── app/
│   ├── config/
│   │   └── config.js      # loads defaults or AWS Secret
│   ├── controller/       # Express route handlers
│   └── views/            # Mustache templates
├── index.js              # Express application entry point
├── package.json
├── code.zip              # lab-provided code bundle
└── README.md             # this file
```

---

## Setup & Deployment

### Phase 2: App Server Provisioning

1. SSH into your Ubuntu EC2 instance.
2. Download and unzip the provided code bundle:
   ```bash
   wget https://.../code.zip -P ~/
   unzip ~/code.zip -d ~/app
   cd ~/app/resources/codebase_partner
   ```
3. Install dependencies:
   ```bash
   sudo apt update && sudo apt install -y nodejs npm unzip mysql-client
   npm install aws-sdk express body-parser cors mustache-express serve-favicon
   ```

### Phase 3: Decoupling & Secrets

1. **Create or update** your Secrets Manager entry named `Mydbsecret`:
   ```bash
   aws secretsmanager create-secret \
     --name Mydbsecret \
     --secret-string '{"user":"nodeapp","password":"student12","host":"<RDS_ENDPOINT>","db":"STUDENTS"}' \
     --region us-east-1
   ```
   If the secret exists, use `update-secret` instead of `create-secret`.

2. **Attach** the IAM `LabInstanceProfile` role to your EC2 so it can retrieve the secret.

3. **Configure** inbound Security Group rules for your web server:
   - HTTP (TCP 80) from `0.0.0.0/0`
   - SSH (TCP 22) from your IP
   - MySQL (TCP 3306) restricted to the RDS SG

---

## Elastic Load Balancer

In Phase 4, you will front your EC2 app servers with an Application Load Balancer (ALB) to distribute traffic and improve fault tolerance:

1. **Create a Target Group**
   - Protocol: HTTP, Port: 80
   - Health check path: `/health` (or `/`)
   - Health check protocol: HTTP
   - Healthy threshold: 2, Unhealthy threshold: 5

2. **Create an Application Load Balancer**
   - Scheme: internet-facing
   - Listeners: HTTP (port 80)
   - Availability Zones: select at least two AZs in your VPC
   - Security group: allow inbound HTTP (80) from `0.0.0.0/0` and outbound to your instance SG

3. **Register Instances**
   - Attach your Auto Scaling group or individual EC2 instances to the target group

4. **Configure Listeners & Rules**
   - Forward incoming HTTP traffic on port 80 to your target group

5. **Test the Load Balancer**
   - Visit the ALB DNS name in your browser: `http://<alb-dns-name>/`
   - Verify requests are spread across multiple instances and health checks are passing

---

## Running the App Locally

By default the app listens on port `3000`. To change it to port `80`:

```bash
# on EC2 (requires sudo to bind <1024):
sudo APP_PORT=80 npm start

# or, permanently modify index.js fallback:
#   const app_port = process.env.APP_PORT || 80;
sudo npm start
```

Verify the listener:

```bash
sudo lsof -iTCP -sTCP:LISTEN -nP | grep LISTEN
```

---

## Accessing the Application

Open in your browser:

```
http://<EC2_PUBLIC_IP>/
```

Perform CRUD on student records via the UI.

---

## Troubleshooting

- **JSON parse error** in `config.js`: ensure your secret’s string is valid JSON—no stray newlines or comments.
- **Port conflicts**: confirm `APP_PORT` matches your SG rule and no OS firewall (e.g., `ufw`) is blocking it.
- **Permission denied** on port 80: run under `sudo` or configure a systemd service.

---

## Cleanup

To avoid lab-budget overages, terminate resources when done:

1. **End Lab** in the AWS Academy interface.
2. Or manually delete the EC2, RDS, ALB, and other resources.

---

> _This README follows the AWS Academy lab “Building a Highly Available, Scalable Web Application.”_

