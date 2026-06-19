# Tooling Website Deployment Automation with Continuous Integration — Jenkins 101

## Project Overview

This project builds on the **Propitix Tooling Website** infrastructure established in earlier projects by bringing in **Continuous Integration (CI)** through Jenkins. In Project 8, a Load Balancer was positioned in front of two Web Servers that both mount shared storage from an NFS Server. Updates were handled manually — every code change required a developer to copy files to the NFS server by hand.

This project does away with manual deployments altogether. A **Jenkins server** is added to the stack and connected to the GitHub repository `https://github.com/cedrick13bienvenue/tooling-jenkins` via a **webhook**. From here on, every `git push` to the repository automatically triggers Jenkins to fetch the latest code and push it to `/mnt/apps` on the NFS Server — instantly refreshing all Web Servers at once, since they both mount that directory.

**Continuous Integration (CI)** is a software development practice that raises the speed and quality of software delivery by having developers commit code in small increments (at least once daily), which is then automatically built and tested before being merged into the shared repository. The aim is to catch integration problems early and ship working software faster.

In this project, Jenkins serves as the CI server that drives the deployment pipeline from code push to live website update — requiring no manual steps after the initial configuration.

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

This project extends the Project 8 infrastructure by introducing a Jenkins Server and a GitHub webhook. The revised deployment flow operates as follows:

1. A developer pushes code to the GitHub `tooling-jenkins` repository
2. GitHub delivers a webhook event to the Jenkins Server
3. Jenkins fetches the latest code and transfers it to `/mnt/apps` on the NFS Server over SSH
4. Both Web Servers, which mount `/mnt/apps` as `/var/www`, immediately begin serving the updated code
5. Client traffic continues to pass through the Load Balancer unchanged

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

All servers from Projects 7 and 8 listed below must be **Running** with **2/2 status checks passed** in the AWS Console before beginning this project:

| Server | Name | Role |
|---|---|---|
| NFS Server | `Project7-NFS` | Shared file storage for Web Servers |
| Web Server 1 | `Project7-Web-1` | Serves the Tooling Website |
| Web Server 2 | `Project7-Web-2` | Serves the Tooling Website |
| DB Server | `Project7-DB` | MySQL database backend |
| Load Balancer | `Project-8-apache-lb` | Routes traffic to Web Servers |

**Prerequisite checklist:**
- All 5 instances listed above show `Running` state with `2/2 checks passed`
- The Tooling Website loads successfully at `http://<LB-PUBLIC-IP>/index.php`
- Both Web Servers have `/var/www` mounted to the NFS Server
- MySQL is active on the DB Server with the `tooling` database and `webaccess` user in place

> **Expected Output**: AWS EC2 Instances list confirming all 5 pre-existing servers are in `Running` state with `2/2 checks passed`.
> ![AWS EC2 console — all existing instances (Project7-NFS, Project7-Web-1, Project7-Web-2, Project7-DB, Project-8-apache-lb) showing Running state with 2/2 status checks passed](screenshoots/all-instances-running.png)

---

## Phase 1: Launch the Jenkins EC2 Instance

### 1.1 Create the EC2 Instance

**1.** Log in to the **AWS Management Console** → go to **EC2** → select **Instances** → click **Launch instances**.

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

> **Note**: Port 8080 is where Jenkins listens by default. It needs to be open so you can reach the Jenkins UI in your browser and so GitHub webhooks can connect to it.

**7.** Under **Configure storage**:

| Field | Value |
|---|---|
| **Root volume size** | `8 GiB` (default) |
| **Volume type** | `gp3` |

**8.** Click **Launch instance**. Then click **View all instances** to return to the list. Wait until:
- **Instance State** = `Running`
- **Status checks** = `2/2 checks passed`

**9.** Click the `Project9-Jenkins` instance row to select it. In the details panel below, copy the **Public IPv4 address** — you will need this throughout the project.

> **Expected Output**: EC2 console showing `Project9-Jenkins` in `Running` state with `2/2 checks passed` and the Public IPv4 address displayed in the details panel.
> ![AWS EC2 console — Project9-Jenkins instance in Running state with 2/2 status checks passed and Public IPv4 address visible in the details panel](screenshoots/jenkins-instance-running.png)

---

## Phase 2: Install Jenkins on the Server

### 2.1 SSH Into the Jenkins Server

**10.** Open a terminal window and go to the directory containing your `.pem` key:

```bash
cd /path/to/your/key
```

**11.** Apply the correct permissions to the key file (SSH requires this):

```bash
chmod 400 cedriq-ec2.pem
```

**12.** SSH into the Jenkins server — swap `<JENKINS-PUBLIC-IP>` for your actual public IP:

```bash
ssh -i cedriq-ec2.pem ubuntu@<JENKINS-PUBLIC-IP>
```

When asked `Are you sure you want to continue connecting (yes/no)?` enter `yes` and press Enter.

> **Note**: The default SSH user for Ubuntu EC2 instances is `ubuntu`.

---

### 2.2 Update the Server

**13.** Update the package index before installing any software:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2.3 Install Java

Jenkins runs on Java, so Java must be installed first. Install OpenJDK 17:

**14.**
```bash
sudo apt install fontconfig openjdk-17-jre -y
```

**15.** Confirm the installation succeeded:

```bash
java -version
```

Expected output:
```
openjdk version "17.0.x" 2024-xx-xx
OpenJDK Runtime Environment (build 17.0.x+x-Ubuntu-...)
OpenJDK 64-Bit Server VM (build 17.0.x+x-Ubuntu-..., mixed mode, sharing)
```

> **Expected Output**: Terminal showing `java -version` with OpenJDK 17 version details.
> ![Terminal — java -version output showing OpenJDK 17 version number](screenshoots/java-version.png)

---

### 2.4 Install Jenkins

**16.** Import the Jenkins repository signing key so Ubuntu can verify the Jenkins packages:

```bash
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys 7198F4B714ABFC68
sudo gpg --export 7198F4B714ABFC68 | sudo tee /usr/share/keyrings/jenkins-keyring.gpg > /dev/null
```

**17.** Register the Jenkins repository in apt sources:

```bash
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

**18.** Refresh the apt package index to include the new Jenkins repository:

```bash
sudo apt update
```

**19.** Install Jenkins:

```bash
sudo apt install jenkins -y
```

---

### 2.5 Start and Enable Jenkins

**20.** Configure Jenkins to start automatically on boot:

```bash
sudo systemctl enable jenkins
```

**21.** Start the Jenkins service:

```bash
sudo systemctl start jenkins
```

**22.** Confirm Jenkins is active:

```bash
sudo systemctl status jenkins
```

Check for the line `Active: active (running)` — it should be highlighted in green. Press `q` to quit.

> **Expected Output**: Terminal showing `sudo systemctl status jenkins` with `Active: active (running)` in green.
> ![Terminal — sudo systemctl status jenkins output showing active (running) in green with the Jenkins process details](screenshoots/jenkins-status-active.png)

---

## Phase 3: Complete the Jenkins Initial Setup in the Browser

### 3.1 Open Jenkins in the Browser

**23.** Open a new browser tab and go to:

```
http://<JENKINS-PUBLIC-IP>:8080
```

> **Note**: Make sure to use the **Public IPv4 address** of your Jenkins instance, not the private IP (`172.31.x.x`). The private IP is only accessible within the AWS VPC and will not load in your browser.

The browser will display a page titled **"Unlock Jenkins"**.

> **Expected Output**: Browser showing the Jenkins "Unlock Jenkins" page with the Administrator password field.
> ![Browser — Jenkins Unlock Jenkins page showing the Administrator password text field and the path to the initialAdminPassword file](screenshoots/jenkins-unlock-page.png)

---

### 3.2 Retrieve the Initial Admin Password

**24.** Return to your terminal (connected to the Jenkins server) and execute:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**25.** Copy the long alphanumeric string that is printed (e.g. `a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4`).

**26.** Paste it into the **Administrator password** field on the Unlock Jenkins page and click **Continue**.

---

### 3.3 Install Suggested Plugins

**27.** On the "Customize Jenkins" screen, click **"Install suggested plugins"** — the left option with the cloud icon.

Jenkins will download and install the standard plugins. This typically takes 3–7 minutes depending on network speed — do not refresh the page.

> **Expected Output**: Jenkins plugin installation progress screen showing multiple plugins with loading bars.
> ![Browser — Jenkins plugin installation progress screen showing multiple plugins being installed with progress indicators](screenshoots/jenkins-plugins-installing.png)

---

### 3.4 Create the First Admin User

**28.** Once plugins have finished installing, complete the **"Create First Admin User"** form:

| Field | Value |
|---|---|
| **Username** | Choose a username (e.g. `admin`) |
| **Password** | Choose a strong password |
| **Confirm password** | Repeat the password |
| **Full name** | Your full name |
| **E-mail address** | Your email |

**29.** Click **"Save and Continue"**.

**30.** On the **"Instance Configuration"** screen, keep the Jenkins URL as pre-populated (`http://<JENKINS-PUBLIC-IP>:8080/`) and click **"Save and Finish"**.

**31.** Click **"Start using Jenkins"**.

This brings you to the Jenkins dashboard — the main interface with the left sidebar containing **New Item**, **People**, **Build History**, and more.

> **Expected Output**: Browser showing the Jenkins main dashboard after first login.
> ![Browser — Jenkins main dashboard after initial setup showing the Welcome to Jenkins screen with the left sidebar navigation](screenshoots/jenkins-dashboard.png)

---

## Phase 4: Connect the GitHub Repository to Jenkins

### 4.1 Prepare the GitHub Repository

**32.** Fork the tooling repository from the Darey.io account on GitHub:

```
https://github.com/darey-io/tooling
```

On the fork page, enter `tooling-jenkins` as the **Repository name**, confirm your GitHub account is the owner, then click **Create fork**.

**33.** After forking, update the repository name if necessary via the repo **Settings** → set the name to `tooling-jenkins` → click **Rename**. Your repository will then be available at:

```
https://github.com/cedrick13bienvenue/tooling-jenkins
```

---

### 4.2 Create a Jenkins Freestyle Job

**34.** On the Jenkins dashboard, click **"New Item"** in the left sidebar.

**35.** Fill in the New Item form:

| Field | Value |
|---|---|
| **Item name** | `tooling-website` |
| **Project type** | `Freestyle project` |

Click **OK**.

---

### 4.3 Configure Source Code Management

**36.** On the job configuration page, navigate to **"Source Code Management"** and choose the **Git** radio button.

**37.** In the **Repository URL** field, enter:

```
https://github.com/cedrick13bienvenue/tooling-jenkins.git
```

> **Note**: The `.git` suffix is required. A red error warning will appear beneath the field — this is normal until credentials are provided.

**38.** Next to the **Credentials** dropdown, click **"+ Add"** → **"Global"** → select **"Username with password"** from the credential type list → click **Next**.

**39.** Fill in the credentials form:

| Field | Value |
|---|---|
| **Kind** | `Username with password` |
| **Username** | `cedrick13bienvenue` |
| **Password** | Your GitHub Personal Access Token (`ghp_...`) |
| **Description** | `GitHub PAT` |

> **Note**: GitHub does not accept your account password for Git operations anymore. A **Personal Access Token (PAT)** is required. To generate one: GitHub → Profile → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic). Enable the `repo` scope and copy the token right away — GitHub only shows it once.

Click **Create**.

**40.** From the **Credentials** dropdown, pick the credentials you just created. The red warning under the Repository URL should clear — indicating Jenkins can access the repository.

**41.** In the **"Branch Specifier"** field, change `*/master` to match your repo's default branch:

```
*/master
```

> **Note**: The `darey-io/tooling` repository defaults to `master`. Confirm the branch name by checking the branch selector on your GitHub repository page.

**42.** Scroll to **"Build Triggers"** and check **"GitHub hook trigger for GITScm polling"**.

**43.** Scroll to **"Build Steps"** → click **"Add build step"** → select **"Execute shell"**. Enter:

```bash
echo "Jenkins build triggered successfully"
ls -la $WORKSPACE
```

**44.** Click **Save**.

> **Expected Output**: Jenkins job configuration page showing the SCM section with the repository URL filled in, credentials selected with no red error, and the branch specifier set.
> ![Browser — Jenkins job config Source Code Management section showing the tooling-jenkins repo URL, credentials selected, and branch specifier with no error message](screenshoots/jenkins-job-scm-config.png)

---

### 4.4 Run the First Manual Build

**45.** On the `tooling-website` job page, click **"Build Now"** in the left sidebar.

**46.** In the **Build History** panel at the lower left, wait for **#1** to complete. A **blue circle** indicates success; a **red circle** means failure.

**47.** Click on **#1** → **"Console Output"** and verify:
- Jenkins cloned your `tooling-jenkins` repository
- The file listing from `ls -la $WORKSPACE` is visible
- The last line reads: `Finished: SUCCESS`

> **Expected Output**: Jenkins Console Output for Build #1 showing the cloned repository files and `Finished: SUCCESS` at the bottom.
> ![Browser — Jenkins Console Output for Build #1 showing repository files cloned from tooling-jenkins and Finished: SUCCESS at the bottom](screenshoots/jenkins-build1-success.png)

---

## Phase 5: Configure GitHub Webhook to Auto-Trigger Jenkins

### 5.1 Add the Webhook in GitHub

**48.** Go to your `tooling-jenkins` repository on GitHub:

```
https://github.com/cedrick13bienvenue/tooling-jenkins
```

**49.** Open the **Settings** tab → select **"Webhooks"** from the left sidebar → click **"Add webhook"**.

**50.** Fill in the webhook form:

| Field | Value |
|---|---|
| **Payload URL** | `http://<JENKINS-PUBLIC-IP>:8080/github-webhook/` |
| **Content type** | `application/json` |
| **Secret** | Leave blank |
| **Which events trigger this webhook?** | `Just the push event` |
| **Active** | Checked |

> **Note**: Do not omit the trailing `/` in the Payload URL. Substitute `<JENKINS-PUBLIC-IP>` with the actual public IP of your `Project9-Jenkins` instance.

**51.** Click **"Add webhook"**.

**52.** GitHub will automatically fire a `ping` event to Jenkins to test the connection. Click the webhook entry, scroll to **"Recent Deliveries"**, and verify that the ping shows a **green checkmark** with **Response code: 200**.

> **Expected Output**: GitHub webhook "Recent Deliveries" showing a ping event with a green checkmark and Response code: 200.
> ![GitHub — webhook Recent Deliveries section showing the ping event with a green checkmark and Response tab displaying Response code 200](screenshoots/github-webhook-ping-success.png)

---

### 5.2 Test the Webhook with a Git Push

**53.** On your local machine, clone the `tooling-jenkins` repository:

```bash
git clone https://github.com/cedrick13bienvenue/tooling-jenkins.git
cd tooling-jenkins
```

**54.** Make a small change to trigger a build:

```bash
echo "# CI/CD pipeline test - Project 9" >> README.md
```

**55.** Stage, commit, and push:

```bash
git add README.md
git commit -m "test jenkins webhook trigger"
git push origin master
```

> **Note**: If prompted for a password during `git push`, enter your GitHub Personal Access Token (`ghp_...`) — not your GitHub account password.

**56.** Switch over to your Jenkins browser tab. A new build should appear in the Build History within 5–10 seconds — kicked off **automatically** by the GitHub webhook, with no manual trigger needed.

**57.** Click on the build number → **"Console Output"**. Read through the output and confirm:
- Jenkins fetched the latest code from `https://github.com/cedrick13bienvenue/tooling-jenkins.git`
- The file listing from `ls -la $WORKSPACE` shows all repo files with current timestamps
- The commit message from your push is visible (e.g. `Commit message: test jenkins webhook trigger`)
- The very last line reads:

```
Finished: SUCCESS
```

> **Expected Output**: Jenkins Console Output showing files fetched from the `tooling-jenkins` repo and `Finished: SUCCESS` at the bottom — triggered automatically by the GitHub webhook.
> ![Browser — Jenkins Console Output showing automatic webhook-triggered build with files fetched from tooling-jenkins and Finished: SUCCESS at the bottom](screenshoots/jenkins-build-auto-triggered.png)

---

## Phase 6: Configure Jenkins to Deploy Files to the NFS Server

Jenkins can now pull code from GitHub automatically. The remaining step is to have Jenkins copy those files to `/mnt/apps` on the NFS Server after each successful build. Since both Web Servers mount `/mnt/apps` as `/var/www`, any update to the NFS Server is immediately reflected on the live website across all Web Servers.

This is handled by the **Publish Over SSH** Jenkins plugin, which copies files from the Jenkins workspace to a remote server via SSH after every build.

---

### 6.1 Install the Publish Over SSH Plugin

**58.** In the Jenkins left sidebar, click **"Manage Jenkins"**.

**59.** Click **"Plugins"** (puzzle piece icon).

**60.** Click the **"Available plugins"** tab.

**61.** In the search box, type `Publish Over SSH`.

**62.** Tick the checkbox next to **"Publish Over SSH"** in the search results.

**63.** Click **"Install"** and wait until all items display **"Success"**.

| Item | Status |
|---|---|
| Infrastructure plugin for Publish Over X | Success |
| JSch dependency | Success |
| Publish Over SSH | Success |
| Loading plugin extensions | Success |

> **Expected Output**: Jenkins plugin download progress page showing all Publish Over SSH dependencies installed with **Success** status.
> ![Browser — Jenkins plugin Download progress page showing Publish Over SSH and all its dependencies installed with Success status](screenshoots/publish-over-ssh-installed.png)

---

### 6.2 Get the NFS Server's Private IP

**64.** Go to **AWS Console** → **EC2** → **Instances** → click on **`Project7-NFS`**.

**65.** In the details panel at the bottom, select the **"Details"** tab and locate the **"Private IPv4 address"** (e.g. `172.31.x.x`). Note it down.

> **Note**: The **private IP** is used here because Jenkins and the NFS server share the same AWS VPC and can communicate directly — without routing traffic over the public internet, which is both faster and more secure.

### 6.3 Allow Jenkins Server to SSH into the NFS Server

For Jenkins to transfer files to the NFS server, the NFS server's Security Group must permit inbound SSH connections from the Jenkins server.

**66.** In the AWS EC2 Instances list, click on **`Project7-NFS`**.

**67.** In the details panel at the bottom, click the **"Security"** tab.

**68.** Click on the Security Group name (e.g. `Project7-NFS-SG`) to open it.

**69.** Click **"Edit inbound rules"** → **"Add rule"** and fill in:

| Field | Value |
|---|---|
| **Type** | `SSH` |
| **Protocol** | `TCP` (auto-filled) |
| **Port range** | `22` (auto-filled) |
| **Source** | Custom → search for and select `Project9-Jenkins-SG` |

> **Note**: Referencing the Jenkins Security Group as the source (rather than a specific IP) means the rule remains valid even if the Jenkins server's public IP changes after a reboot.

**70.** Click **"Save rules"**.

---

### 6.4 Configure the SSH Connection to the NFS Server in Jenkins

**71.** In Jenkins, click **"Manage Jenkins"** → **"System"** (wrench icon).

**72.** Scroll to the bottom of the page to find the **"Publish over SSH"** section.

**73.** Under **"SSH Servers"**, click **"Add"**. A form expands — fill it in:

| Field | Value |
|---|---|
| **Name** | `NFS-Server` |
| **Hostname** | Private IP of `Project7-NFS` (e.g. `172.31.x.x`) |
| **Username** | `ec2-user` |
| **Remote Directory** | `/mnt/apps` |

> **Note**: The NFS server runs on RHEL, where the default SSH user is `ec2-user`, not `ubuntu`.

**74.** Click **"Advanced"** (small link below the fields):
- Check **"Use password authentication, or use a different key"**
- A **Key** text area will appear — open a terminal and run:

```bash
cat /path/to/cedriq-ec2.pem
```

- Copy the entire output from `-----BEGIN RSA PRIVATE KEY-----` to `-----END RSA PRIVATE KEY-----` and paste it into the **Key** field.

**75.** Click **"Test Configuration"**.

This should return **"Success"**. If it does not:

| Error | Fix |
|---|---|
| `Auth fail` | Double-check Username is `ec2-user` and the full key was pasted including header and footer lines |
| `Connection refused` / `timed out` | The NFS security group inbound rule from Step 6.3 was not saved correctly — re-check it |

> **Expected Output**: Jenkins "Configure System" page showing the Publish over SSH section with NFS-Server config filled in and **"Success"** after clicking "Test Configuration".
> ![Browser — Jenkins Configure System page showing Publish over SSH section with NFS-Server hostname, ec2-user, /mnt/apps filled in and Test Configuration showing Success](screenshoots/jenkins-ssh-config-success.png)

**76.** Click **"Save"** at the bottom of the page.

---

### 6.5 Update the Jenkins Job to Deploy Files to NFS

**77.** In Jenkins, click on the **`tooling-website`** job → click **"Configure"** in the left sidebar.

**78.** Scroll to **"Build Steps"**. In the Execute Shell text area, replace the existing content with:

```bash
echo "Build workspace: $WORKSPACE"
```

**79.** Scroll further down to **"Post-build Actions"** → click **"Add post-build action"** → select **"Send build artifacts over SSH"**.

**80.** Fill in the SSH transfer section:

| Field | Value |
|---|---|
| **SSH Server** | `NFS-Server` (select from dropdown) |
| **Source files** | `**` |
| **Remove prefix** | Leave blank |
| **Remote directory** | Leave blank (defaults to `/mnt/apps`) |
| **Exec command** | Leave blank |

> **Note**: `**` means "transfer every file and folder from the Jenkins workspace". This copies the entire contents of the `tooling-jenkins` repo to `/mnt/apps` on the NFS server after every successful build.

**81.** Click **"Save"**.

> **Expected Output**: Jenkins job configuration page showing the Post-build Actions section with "Send build artifacts over SSH" configured — NFS-Server selected and `**` in the Source files field.
> ![Browser — Jenkins tooling-website job Post-build Actions section showing Send build artifacts over SSH with NFS-Server selected and ** in Source files](screenshoots/jenkins-job-postbuild-config.png)

---

### 6.6 Set Correct Permissions on the NFS Server

Jenkins connects to the NFS server as `ec2-user` and writes files to `/mnt/apps`. If `ec2-user` does not have ownership of that directory, all builds will fail with a **"Permission denied"** error. Resolve this before running the pipeline.

**82.** Connect to the NFS server from your terminal:

```bash
ssh -i /path/to/cedriq-ec2.pem ec2-user@<NFS-PUBLIC-IP>
```

**83.** Inspect the current ownership of `/mnt/apps`:

```bash
ls -la /mnt/
```

The output will likely show `/mnt/apps` owned by `root`:

```
drwxr-xr-x  2 root     root     4096 ...  apps
```

**84.** Transfer ownership to `ec2-user` so it can write to the directory:

```bash
sudo chown -R ec2-user:ec2-user /mnt/apps
```

**85.** Confirm the ownership has been updated:

```bash
ls -la /mnt/
```

Expected output:

```
drwxr-xr-x  2 ec2-user ec2-user 4096 ...  apps
```

> **Expected Output**: NFS server terminal showing `ls -la /mnt/` with `/mnt/apps` owned by `ec2-user`.
> ![Terminal — NFS server showing ls -la /mnt/ output with /mnt/apps directory owned by ec2-user ec2-user](screenshoots/nfs-apps-permissions.png)

**86.** Run `exit` to disconnect from the NFS server and return to your local terminal.

---

## Phase 7: End-to-End Test — Push Code and Watch the Full Pipeline Run

This final test confirms that the complete CI/CD pipeline is functioning. A single `git push` from your local machine should cause Jenkins to automatically retrieve the code and deploy it to the NFS server — with no manual intervention required.

### 7.1 Push a Code Change

**87.** From your local machine, navigate to your `tooling-jenkins` folder:

```bash
cd /path/to/tooling-jenkins
```

**88.** Append a comment to the main PHP file to verify the deployment landed:

```bash
echo "<!-- Deployed by Jenkins - Project 9 -->" >> html/index.php
```

**89.** Stage, commit, and push:

```bash
git add html/index.php
git commit -m "add jenkins deployment comment to index.php"
git push origin master
```

---

### 7.2 Watch Jenkins React Automatically

**90.** Jump to your Jenkins browser tab right after pushing.

**91.** A new build should appear in the Build History within 5–10 seconds — automatically fired by the GitHub webhook.

**92.** Click on the build → **"Console Output"**. Read through it carefully and confirm all three stages completed:

**Stage 1 — Code checkout:**
```
Fetching upstream changes from https://github.com/cedrick13bienvenue/tooling-jenkins.git
```

**Stage 2 — Build step:**
```
Build workspace: /var/lib/jenkins/workspace/tooling-website
```

**Stage 3 — SSH file transfer:**
```
SSH: Transferred XX file(s)
Finished: SUCCESS
```

> **Expected Output**: Jenkins Console Output showing all three stages — Git checkout, build step, and `SSH: Transferred XX file(s)` — followed by `Finished: SUCCESS`.
> ![Browser — Jenkins Console Output for the auto-triggered build showing SSH: Transferred XX file(s) and Finished: SUCCESS at the bottom](screenshoots/jenkins-deploy-console-success.png)

### 7.3 Verify Files Arrived on the NFS Server

**93.** Connect to the NFS server:

```bash
ssh -i /path/to/cedriq-ec2.pem ec2-user@<NFS-PUBLIC-IP>
```

**94.** List the contents of `/mnt/apps` and review the timestamps — they should reflect the current time:

```bash
ls -la /mnt/apps/
```

**95.** Verify that your specific change was deployed:

```bash
tail -5 /mnt/apps/html/index.php
```

The comment should appear at the end of the file:

```
<!-- Deployed by Jenkins - Project 9 -->
```

> **Expected Output**: NFS server terminal showing `ls -la /mnt/apps/` with today's timestamps and `tail -5 /mnt/apps/html/index.php` showing the deployed comment.
> ![Terminal — NFS server showing ls -la /mnt/apps/ with current timestamps and tail of index.php showing the Jenkins deployment comment](screenshoots/nfs-files-deployed.png)

**96.** Run `exit` to disconnect from the NFS server.

---

### 7.4 Confirm the Live Website

**97.** Open a browser and access the Tooling Website via the Load Balancer:

```
http://<LB-PUBLIC-IP>/index.php
```

Use the public IP of your `Project-8-apache-lb` instance from the AWS Console in place of `<LB-PUBLIC-IP>`.

**98.** The Tooling Website login page should appear — verifying that the full pipeline is operational:
- GitHub webhook triggered Jenkins on push ✓
- Jenkins pulled the latest code ✓
- Jenkins deployed it to `/mnt/apps` on the NFS server ✓
- Both Web Servers (mounting `/mnt/apps`) served the updated code ✓
- The Load Balancer routed traffic correctly ✓

> **Expected Output**: Browser showing the Tooling Website login page loaded via the Load Balancer public IP.
> ![Browser — Tooling Website login page loaded successfully via the Load Balancer URL confirming the full CI/CD pipeline is working](screenshoots/tooling-website-live.png)

