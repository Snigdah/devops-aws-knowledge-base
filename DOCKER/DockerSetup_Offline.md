# Docker Offline Installation Guide for Ubuntu 24.04 (Noble Numbat)

## System Information
cat /etc/os-release

PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo

## Download Link
https://download.docker.com/linux/ubuntu/dists/noble/pool/stable/amd64/

## Required Packages
- `containerd.io_1.7.29-1~ubuntu.24.04~noble_amd64.deb`
- `docker-ce-cli_29.2.1-1~ubuntu.24.04~noble_amd64.deb`
- `docker-ce_29.2.1-1~ubuntu.24.04~noble_amd64.deb`
- `docker-buildx-plugin_0.31.1-1~ubuntu.24.04~noble_amd64.deb`
- `docker-compose-plugin_2.40.3-1~ubuntu.24.04~noble_amd64.deb`

## Installation Steps

### Step 0: Download from this link
Download all 5 .deb files from the link above

### Step 1: Install in correct order
Run these one by one:

`sudo dpkg -i containerd.io_1.7.29-1~ubuntu.24.04~noble_amd64.deb`

`sudo dpkg -i docker-ce-cli_29.2.1-1~ubuntu.24.04~noble_amd64.deb`

`sudo dpkg -i docker-ce_29.2.1-1~ubuntu.24.04~noble_amd64.deb`

`sudo dpkg -i docker-buildx-plugin_0.31.1-1~ubuntu.24.04~noble_amd64.deb`

`sudo dpkg -i docker-compose-plugin_2.40.3-1~ubuntu.24.04~noble_amd64.deb`

### Step 2: If ANY error appears

`sudo dpkg -i *.deb`

### Step 3: Start Docker

`sudo systemctl daemon-reexec`

`sudo systemctl start docker`

`sudo systemctl enable docker`

### Step 4: Verify installation

`docker --version`