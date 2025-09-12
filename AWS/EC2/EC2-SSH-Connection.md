# EC2 Connection

![](https://media.licdn.com/dms/image/v2/D5612AQGxlv_dxr7UVA/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1718606659488?e=1760572800&v=beta&t=3Irabc3Em0A-UGrgVKOwlXD2mhSP7JET7lNcUIMUk94)

## 1. Traditional PEM Key Authentication

```yaml
ssh -i your-key.pem ec2-user@your-instance-ip
```

## 2. Enable root login and password authentication

```bash
# Switch to root
sudo su

# Set root password
passwd root

# Unlock root account if locked
passwd -u root

# Edit SSH configuration
nano /etc/ssh/sshd_config

# Required configuration:
PermitRootLogin yes
PasswordAuthentication yes

# Restart SSH service
systemctl restart sshd
```

### Enable Password Authentication (Ubuntu)

Check if disabled:

```bash
grep -i PasswordAuthentication /etc/ssh/sshd_config.d/* 2>/dev/null
```

If you see PasswordAuthentication no, edit the config file:

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf

# Change the line to:
PasswordAuthentication yes

# Restart SSH service
sudo systemctl restart ssh
```

## 3. SSH Key-Based (Passwordless) Authentication

**Generate Key Pair on Source Server**

```bash
ssh-keygen -t rsa
cd ~/.ssh
ls -ltr
cat id_rsa.pub
```

**Configure Target Server**

```bash
# Add public key to authorized_keys
nano ~/.ssh/authorized_keys
# Paste public key content and save
```

**Test Connection**

```bash
ssh root@target-server-ip
```

## ⚡ Quick Setup Commands

```bash
# One-line key generation (no passphrase)
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# Copy key to remote server (if ssh-copy-id available)
ssh-copy-id root@target-server-ip

# Fix permissions (critical for SSH)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
```
