# Tooling Website Deployment Automation with Continuous Integration — Jenkins 101

## Project Overview

This project extends the **Propitix Tooling Website** infrastructure built in previous projects by introducing **Continuous Integration (CI)** using Jenkins. In the previous project (Project 8), a Load Balancer was placed in front of two Web Servers that both mount shared storage from an NFS Server. Deployments were done manually — any code update required a developer to manually copy files to the NFS server.

This project eliminates manual deployments entirely. A **Jenkins server** is added to the infrastructure and connected to the GitHub repository `https://github.com/cedrick13bienvenue/tooling-jenkins` via a **webhook**. From this point forward, every `git push` to the repository automatically triggers Jenkins to pull the latest code and deploy it directly to `/mnt/apps` on the NFS Server — which instantly updates all Web Servers simultaneously, since they both mount that directory.

**Continuous Integration (CI)** is a software development strategy that increases the speed and quality of software delivery by having developers commit code in small increments (at least daily), which is then automatically built and tested before it is merged with the shared repository. The goal is to catch integration issues early and deliver working software faster.

In this project, Jenkins acts as the CI server that automates the deployment pipeline from code push to live website update — with zero manual intervention after the initial setup.

---

## Technologies Used

| Component | Details |
|---|---|
| Infrastructure | AWS EC2 |
| Jenkins Server OS | Ubuntu Server 24.04 LTS |
| Web Server OS | Red Hat Enterprise Linux 8 |
| Database OS | Ubuntu Server 24.04 LTS |
| NFS Server OS | Red Hat Enterprise Linux 8 |
| Load Balancer OS | Ubuntu Server 24.04 LTS |
| CI Server | Jenkins 2.x (LTS) |
| Load Balancer Software | Apache2 (`mod_proxy_balancer`) |
| Web Server Software | Apache (`httpd`) + PHP |
| Database | MySQL |
| Version Control | Git + GitHub |
| Deployment Method | Publish Over SSH (Jenkins plugin) |
| Code Repository | `https://github.com/cedrick13bienvenue/tooling-jenkins` |

---

## Architecture

This project adds a Jenkins Server and a GitHub webhook to the existing Project 8 infrastructure. The updated deployment flow works as follows:

1. A developer pushes code to the GitHub `tooling-jenkins` repository
2. GitHub sends a webhook notification to the Jenkins Server
3. Jenkins pulls the latest code and copies it to `/mnt/apps` on the NFS Server via SSH
4. Both Web Servers (which mount `/mnt/apps` as `/var/www`) immediately serve the updated code
5. Client traffic continues to flow through the Load Balancer as before

```
          GitHub Repository
https://github.com/cedrick13bienvenue/tooling-jenkins
                |
             Webhook
                |
         Jenkins Server              ← Ubuntu 24.04 (Jenkins 2.x)
       <JENKINS-PUBLIC-IP>
                |
            TCP 22 (SSH Deploy)
                |
           NFS Server                ← RHEL 8 (/mnt/apps)
         <NFS-PRIVATE-IP>
         /              \
  TCP/UDP 2049        TCP/UDP 2049
  UDP 111             UDP 111
       /                    \
 Web-Server-1          Web-Server-2  ← RHEL 8 (Apache httpd + PHP)
      |                      |
      └──────────┬────────────┘
             TCP 3306
                 |
            DB Server               ← Ubuntu 24.04 (MySQL)
          <DB-PRIVATE-IP>

 Web-Server-1          Web-Server-2
      \                      /
       \                    /
        TCP 80          TCP 80
              \        /
           Load Balancer             ← Ubuntu 24.04 (Apache2)
         <LB-PUBLIC-IP>
                |
             TCP 80
                |
             Client
```

**Traffic types:**
| Traffic | Path |
|---|---|
| Client traffic | Client → Load Balancer → Web Servers |
| DB traffic | Web Servers → DB Server (TCP 3306) |
| NFS traffic | Web Servers ↔ NFS Server (TCP/UDP 2049, 111) |
| Deploy traffic | Jenkins Server → NFS Server (TCP 22) |

---

## Prerequisites

The following servers from Projects 7 and 8 must be **Running** with **2/2 status checks passed** in your AWS Console before starting this project:

| Server | Name | Role |
|---|---|---|
| NFS Server | `Project7-NFS` | Shared file storage for Web Servers |
| Web Server 1 | `Project7-Web-1` | Serves the Tooling Website |
| Web Server 2 | `Project7-Web-2` | Serves the Tooling Website |
| DB Server | `Project7-DB` | MySQL database backend |
| Load Balancer | `Project-8-apache-lb` | Routes traffic to Web Servers |

**Prerequisite checklist:**
- All 5 instances above are in `Running` state with `2/2 checks passed`
- The Tooling Website loads correctly at `http://<LB-PUBLIC-IP>/index.php`
- Both Web Servers have `/var/www` mounted from the NFS Server
- MySQL is running on the DB Server with the `tooling` database and `webaccess` user intact

> **Expected Output**: AWS EC2 Instances list showing all 5 existing servers in `Running` state with `2/2 checks passed`.
> ![AWS EC2 console — all existing instances (Project7-NFS, Project7-Web-1, Project7-Web-2, Project7-DB, Project-8-apache-lb) showing Running state with 2/2 status checks passed](screenshoots/all-instances-running.png)

---

## Phase 1: Launch the Jenkins EC2 Instance

### 1.1 Create the EC2 Instance

**1.** Sign in to the **AWS Management Console** → navigate to **EC2** → click **Instances** → click **Launch instances**.

**2.** Under **Name and tags**:

| Field | Value |
|---|---|
| **Name** | `Project9-Jenkins` |

**3.** Under **Application and OS Images (Amazon Machine Image)**:

| Field | Value |
|---|---|
| **AMI** | Click **"Ubuntu"** from the quick-select tabs |
| **Version** | `Ubuntu Server 24.04 LTS (HVM), SSD Volume Type` |
| **Architecture** | `64-bit (x86)` |

**4.** Under **Instance type**:

| Field | Value |
|---|---|
| **Instance type** | `t2.micro` (Free tier eligible). Use `t3.micro` if unavailable. |

**5.** Under **Key pair (login)**:

| Field | Value |
|---|---|
| **Key pair name** | Select your existing `.pem` key pair |

**6.** Under **Network settings** → click **Edit**:

| Field | Value |
|---|---|
| **VPC** | Default VPC |
| **Subnet** | Leave as default |
| **Auto-assign public IP** | `Enable` |
| **Firewall** | Select **"Create security group"** |
| **Security group name** | `Project9-Jenkins-SG` |
| **Description** | `Security group for Jenkins CI server` |

**Inbound Rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| `SSH` | `TCP` | `22` | `My IP` |
| `Custom TCP` | `TCP` | `8080` | `Anywhere-IPv4` (0.0.0.0/0) |

> **Note**: Port 8080 is the default port Jenkins listens on. It must be open so you can access the Jenkins UI from your browser and so GitHub webhooks can reach it.

**7.** Under **Configure storage**:

| Field | Value |
|---|---|
| **Root volume size** | `8 GiB` (default) |
| **Volume type** | `gp3` |

**8.** Click **Launch instance**. Click **View all instances** to return to the instances list. Wait until:
- **Instance State** = `Running`
- **Status checks** = `2/2 checks passed`

**9.** Click on the `Project9-Jenkins` instance row to select it. In the details panel at the bottom, note down the **Public IPv4 address** — you will use this throughout the project.

> **Expected Output**: EC2 console showing `Project9-Jenkins` in `Running` state with `2/2 checks passed` and the Public IPv4 address visible in the details panel.
> ![AWS EC2 console — Project9-Jenkins instance in Running state with 2/2 status checks passed and Public IPv4 address visible in the details panel](screenshoots/jenkins-instance-running.png)
