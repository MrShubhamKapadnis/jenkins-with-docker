# Jenkins on Docker Setup with Persistent Storage

<p align="center">
  <img src="https://www.jenkins.io/images/logos/jenkins/jenkins.png" width="200" alt="Jenkins Logo">
</p>

## Jenkins on Docker Setup 🚀

This repository contains a **comprehensive guide** to setting up **Jenkins in Docker** with persistent storage and production-grade practices. It serves as a knowledge transfer document for team members, DevOps engineers, or anyone interested in setting up Jenkins using Docker.

## 📄 Document Summary

The main document (`jenkins_docker_setup.md`) provides:

- ✅ Step-by-step **installation** using Docker and Docker Compose  
- 💾 Setup for **persistent data storage** on mounted volumes  
- 🔐 Configurations for **security**, **backups**, **restore**, and **plugin management**  
- 🧰 Ready-to-use **scripts** for health checks, auto-restart, and emergency recovery  
- 🛠️ **Troubleshooting tips**, operational commands, and best practices  
- 📋 **Knowledge Transfer Checklist** to help new team members ramp up quickly  

---

## 🧱 Tech Stack

- Jenkins (LTS)
- Docker & Docker Compose
- Linux (Tested on CentOS, Ubuntu)
- Persistent volume: `/home/jenkins_data`
- Ports: `8080` (Web), `50000` (Agent)

---

## 🛠️ Quick Start

```bash
# Clone this repository (or access the document directly if committed in Jenkins)
cd /home/jenkins_data

# Start Jenkins using Docker Compose
docker compose up -d
#or
#docker-compose up -d (if older version)


Shubham Kapadnis
Cloud & DevOps Engineer | Infrastructure Automation | Cloud-Native Enthusiast
