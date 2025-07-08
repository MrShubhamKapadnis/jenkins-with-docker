# Jenkins on Docker Setup with Persistent Storage

![Jenkins Logo](https://www.jenkins.io/images/logos/jenkins/jenkins.png)  
[![Docker Pulls](https://img.shields.io/docker/pulls/jenkins/jenkins)](https://hub.docker.com/r/jenkins/jenkins/)
[![GitHub License](https://img.shields.io/github/license/your-org/jenkins-docker-kt)](LICENSE)
[![Documentation Status](https://readthedocs.org/projects/jenkins-docker-kt/badge/)](docs/README.md)

Jenkins setup using Docker to build and run CI/CD pipelines efficiently. Ideal for DevOps automation and local development.

Production-ready Jenkins deployment in Docker with:
- Persistent storage on mounted disks (`/home`)
- Configuration-as-Code (JCasC) support
- Automated backups
- Security best practices
- Complete knowledge transfer documentation

## 📌 Quick Start

```bash
# Clone repository
git clone https://github.com/your-org/jenkins-docker-kt.git
cd jenkins-docker-kt

# Configure environment
cp .env.sample .env
nano .env  # Edit with your values

# Deploy Jenkins
docker compose -f configs/docker-compose.yml up -d

# Access Jenkins
echo "Open http://$(hostname -I | awk '{print $1}'):8080"
