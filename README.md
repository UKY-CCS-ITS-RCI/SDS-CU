# SDS Deployment Guide

This guide explains how to set up and run **SDS** on a Linux system using Docker.  
Follow each step in order — starting with Docker installation, then user setup, firewall configuration, cloning repositories, and running the container.

---

## Step 0 – Ensure Docker Is Installed

Run these commands as **root**:

```bash
# Install Docker (choose one based on your system)
dnf install -y docker
# or
yum install -y docker

# Enable and start Docker service
systemctl enable docker
systemctl start docker

# Verify Docker is running
docker --version
systemctl status docker
```

---

## Step 1 – Create user 'sds'

```bash
# Create a new user for SDS
adduser sds
passwd sds
```

---

## Step 2 – Run as root

```bash
# Open required ports for HTTP (80) and app traffic (8080)
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# Allow passwordless docker for user 'sds'
echo "sds ALL=(ALL) NOPASSWD:/usr/bin/docker" > /etc/sudoers.d/sds
chmod 440 /etc/sudoers.d/sds

# Add 'sds' to docker group
usermod -aG docker sds

# Reload firewall and verify
firewall-cmd --reload
firewall-cmd --list-ports
```

---

## Step 3 – Switch to user 'sds'

```bash
su - sds

# Clone the SDS repository
git clone https://github.com/access-ci-org/SDS-Public.git
cd SDS-Public/

# Download the configuration file
wget https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/config.yaml

# Edit config.yaml to add your API key
nano config.yaml
```

---

## Step 4 – Prepare spider_data directories

```bash
mkdir spider_data
cd spider_data

# Create directories and download module files
mkdir alpine
wget -O alpine/alpine_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/alpine_modulespider.out

mkdir blanca
wget -O blanca/blanca_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/blanca_modulespider.out

# Return to main SDS-Public directory
cd ~/SDS-Public/
```

---

## Step 5 – Build and start Docker containers

```bash
# Build and start in detached (daemon) mode
sudo docker compose up --build -d

# Verify running containers
sudo docker ps
```

---

## Step 6 – Rebuild or restart after config changes

If you modify `config.yaml` or other files:

```bash
# Stop and remove running containers
sudo docker compose down

# Rebuild and start again in detached mode
sudo docker compose up --build -d
```

For quick stop/start (no rebuild):

```bash
sudo docker compose stop
sudo docker compose start
```

---

✅ **That’s it!**  
Your SDS service should now be running and accessible on ports **80** and **8080**.
