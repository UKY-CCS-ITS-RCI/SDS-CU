# SDS Deployment Guide @ CU

This guide explains how to set up and run **SDS** on a Linux system using Docker.  
Follow each step in order — starting with Docker installation, then user and firewall setup, cloning repositories, and running the container.
For the bare minimum setup see https://github.com/access-ci-org/SDS-Public/tree/stand-alone  
For more detailed customization see [https://github.com/access-ci-org/SDS-Public/tree/stand-alone ](https://github.com/access-ci-org/SDS-Public/blob/stand-alone/SDS_SETUP.md)  

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

## Step 1 – Create user 'sds' and configure Docker access (as root)

```bash
# Create a new user for SDS
adduser sds
passwd sds

# Add 'sds' to docker group
usermod -aG docker sds

# Allow passwordless docker for user 'sds'
echo "sds ALL=(ALL) NOPASSWD:/usr/bin/docker" > /etc/sudoers.d/sds
chmod 440 /etc/sudoers.d/sds

# Open required ports for HTTP (80) and app traffic (8080)
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# Verify firewall settings
firewall-cmd --list-ports
```

---

## Step 2 – Switch to user 'sds'

```bash
su - sds

# Pull container
docker image pull public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# Download the configuration file
wget https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/config.yaml

# Edit config.yaml to add your API key
nano config.yaml
```

---

## Step 3 – Prepare spider_data and other directories

```bash

mkdir spider_data container_data software_uses websites logs analytics

cd spider_data

# Create directories and download module files
mkdir alpine
wget -O alpine/alpine_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/alpine_modulespider.out

mkdir blanca
wget -O blanca/blanca_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/blanca_modulespider.out

# Return to main directory
cd ..
```

---

## Step 4 – Run the container

```bash
# Start container with minimal mounts
docker run -d -p 8080:80 --mount type=bind,source="./config.yaml",target="/sds/config.yaml" -v ./spider_data:/sds/spider_data --name sds public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# Or start container and mount minimal and other helpful directories (RECOMMENDED)
docker run -d -p 8080:80 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# The other directories created in step 3 can also be mounted if used (mount them to `/sds/<folder_name>` in the container)

# Verify running containers
docker ps -a
```

---

## Step 5 – Stopping and deleting containers

**The SDS will automatically rebuild/re-run if changes are made to the following locations: `config.yaml` , `spider_data` (and a few others)**  

```bash
# Stop running containers
docker stop sds

# Start stopped containers
docker start sds

# Remove existing containers
docker rm sds

# if a container is removed you will have to rerun it using the command in step 4

```

---

**That’s it!**  
Your SDS service should now be running and accessible on ports **8080**.

---
## Enabling HTTPS/SSL

First make sure port 443 is open on your machine:
```
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# Verify firewall settings
firewall-cmd --list-ports
```

Create a `ssl` directory and add your keys into it

```
# make .ssl directory
mkdir ssl
# add you ssl certificate (cert.pem) and key (key.pem) to the ssl directory

```

Create a nginx config file
```
cat << EOF > nginx.conf
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    access_log /var/log/supervisor/nginx-access.log;

    client_max_body_size 100M;
}
EOF
```

Stop and re-run the container with the following changes:
 - Add the appropriate ports (change -p 8080:80 to -p 443:443)
 - Mount the new config to the container (--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default")

```
# stop and remove the container
docker stop sds && docker rm sds

# rerun the container with the appropriate port and mounts
# sudo permissions may be required for exposing port 443
sudo docker run -d -p 443:443 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default"
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest
```

