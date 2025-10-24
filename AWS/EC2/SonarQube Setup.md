# SonarQube Setup on Amazon Linux 2

![](https://buddy.works/guides/covers/sonarqube/sonarqube-share.png)

## Prerequisites

Amazon Linux 2 EC2 instance

Java 11 or later

Minimum 2GB RAM

## 1. Install Java

```yaml
# Install Java 21 (SonarQube requirement)
sudo yum install java-21-amazon-corretto-headless
sudo yum install -y java-21-amazon-corretto-devel

# Verify Java installation
java -version
javac -version
```

## 2. Download and Install SonarQube

```bash
# Navigate to /opt directory
cd /opt

# Download SonarQube community edition
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-25.9.0.112764.zip

# Unzip the downloaded package
sudo unzip sonarqube-25.9.0.112764.zip

# Rename directory for simplicity
cd sonarqube-25.9.0.112764
```

## 3. Create Dedicated User (Security Best Practice)

```bash
# Create system user for SonarQube
sudo useradd sonaradmin

# Change ownership of SonarQube directory
sudo chown -R sonaradmin:sonaradmin sonarqube-25.9.0.112764

#Elasticsearch won't run as root
```

## 4. Directory Structure Overview

```bash
# Explore SonarQube directory structure
cd /opt/sonarqube
ls -ltr

# Key directories:
# bin/     - Startup and control scripts
# conf/    - Configuration files
# logs/    - Application logs
# data/    - Database and stored data
# web/     - Web application files
```

## 5. Start SonarQube Service

```bash
# Switch to sonaradmin user (important: cannot run as root)
sudo su - sonaradmin

# Navigate to binary directory
cd /opt/sonarqube/bin/linux-x86-64/

# Start SonarQube service
./sonar.sh start

# Check service status
./sonar.sh status
```

## 6. Check Logs for Errors

```bash
# If startup fails, check logs
tail -f /opt/sonarqube/logs/sonar.log
tail -f /opt/sonarqube/logs/web.log

# Check Port
sudo netstat -tulnp | grep 9000
```
