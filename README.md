# Tooling Website Deployment Automation with Continuous Integration — Jenkins 101

## Project Overview

This project extends the **Propitix Tooling Website** infrastructure built in previous projects by introducing **Continuous Integration (CI)** using Jenkins. In the previous project (Project 8), a Load Balancer was placed in front of two Web Servers that both mount shared storage from an NFS Server. Deployments were done manually — any code update required a developer to manually copy files to the NFS server.

This project eliminates manual deployments entirely. A **Jenkins server** is added to the infrastructure and connected to the GitHub repository `https://github.com/cedrick13bienvenue/tooling-jenkins` via a **webhook**. From this point forward, every `git push` to the repository automatically triggers Jenkins to pull the latest code and deploy it directly to `/mnt/apps` on the NFS Server — which instantly updates all Web Servers simultaneously, since they both mount that directory.

**Continuous Integration (CI)** is a software development strategy that increases the speed and quality of software delivery by having developers commit code in small increments (at least daily), which is then automatically built and tested before it is merged with the shared repository. The goal is to catch integration issues early and deliver working software faster.

In this project, Jenkins acts as the CI server that automates the deployment pipeline from code push to live website update — with zero manual intervention after the initial setup.
