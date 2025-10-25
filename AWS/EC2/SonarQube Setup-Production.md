# SonarQube Production Setup on Amazon Linux 2

![](https://i.ytimg.com/vi/rdmYgr2Z_io/maxresdefault.jpg)

## Prerequisites

Amazon Linux 2 EC2 instance

Java 11 or later

Minimum 2GB RAM

## ⚙️ 1. System Setup & Docker Installation

```yaml
# Update system
sudo dnf update -y

# Install Docker
sudo dnf install docker -y

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add ec2-user to Docker group (to run docker without sudo)
sudo usermod -aG docker ec2-user
newgrp docker


# Verify Docker installation
docker --version
```

## 2. Configure System Limits & Install Docker Compose

```bash
# Increase virtual memory mapping (required by SonarQube)
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Install Docker Compose plugin (latest version)
sudo mkdir -p /usr/libexec/docker/cli-plugins
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)" -o /usr/libexec/docker/cli-plugins/docker-compose
sudo chmod +x /usr/libexec/docker/cli-plugins/docker-compose

# Verify installation
docker compose version
```

## 3. Deploy SonarQube

```bash
# Create directory
mkdir sonarqube
cd sonarqube

# Create and edit the file with nano or vim
nano docker-compose.yml

or

vim docker-compose.yml

# Check if file exists
ls -la docker-compose.yml

# View the file content to make sure it's correct
cat docker-compose.yml

# Start services
docker compose up -d

# Check status
docker compose ps

# View the logs (wait for "SonarQube is operational")
docker compose logs -f sonarqube
```

## 4. docker-compose.yml

```bash
version: "3.8"

services:
  postgres:
    image: postgres:15
    container_name: sonardb
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - sonardb_data:/var/lib/postgresql/data
    networks:
      - sonarnet

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://postgres:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    networks:
      - sonarnet

volumes:
  sonardb_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:

networks:
  sonarnet:
    driver: bridge
```
