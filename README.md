# SDS Deployment Guide @ CU

This guide explains how to set up and run **SDS** on a Linux system using Podman.  
Follow each step in order — starting with Podman installation, then user and firewall setup, cloning repositories, and running the container.
For the bare minimum setup see https://github.com/access-ci-org/SDS-Public/tree/stand-alone  
For more detailed customization see [https://github.com/access-ci-org/SDS-Public/tree/stand-alone ](https://github.com/access-ci-org/SDS-Public/blob/stand-alone/SDS_SETUP.md)

## Table of Contents

1. [Step 0 – Ensure Podman Is Installed](#step-0--ensure-podman-is-installed)
2. [Step 1 – Create User 'sds' and Configure Podman Access](#step-1--create-user-sds-and-configure-podman-access)
3. [Step 2 – Pull Image](#step-2--pull-image)
4. [Step 3 – Prepare `spider_data` and Other Directories](#step-3--prepare-spider_data-and-other-directories)
5. [Step 4 – Run the Container](#step-4--run-the-container)
6. [Step 5 – Stopping and Deleting Containers](#step-5--stopping-and-deleting-containers)
7. [Updating SDS](#updating-sds)
8. [Enabling HTTPS/SSL](#enabling-httpsssl)
9. [Custom or No Example Use](#custom-or-no-example-use)

---

## Step 0 – Ensure Podman is installed
This should already be done on your system image

---

## Step 1 – Create user 'sds' and configure Podman access

```bash
# Create a new user for SDS
adduser sds
passwd sds

# Allow passwordless podman for user 'sds'
echo "sds ALL=(ALL) NOPASSWD:/usr/bin/podman" > /etc/sudoers.d/sds
chmod 440 /etc/sudoers.d/sds

# Open required ports for HTTP (80) and app traffic (8080)
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# Verify firewall settings
firewall-cmd --list-ports
```

---

## Step 2 – Pull image

```bash
su - sds

# Pull container
podman image pull public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

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
# podman run -d -p 8080:80 --mount type=bind,source="./config.yaml",target="/sds/config.yaml" -v ./spider_data:/sds/spider_data --name sds public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# Or start container and mount minimal and other helpful directories (RECOMMENDED)
podman run -d -p 8080:80 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# The other directories created in step 3 can also be mounted if used (mount them to `/sds/<folder_name>` in the container)

# Verify running containers
podman ps -a
```

---

## Step 5 – Stopping and deleting containers

**The SDS will automatically rebuild/re-run if changes are made to the following locations: `config.yaml` , `spider_data` (and a few others)**  

```bash
# Stop running containers
podman stop sds

# Start stopped containers
podman start sds

# Remove existing containers
podman rm sds

# if a container is removed you will have to rerun it using the command in step 4

```

---

**That’s it!**  
Your SDS service should now be running and accessible on ports **8080**.

---

## Updating SDS
Stop and remove the current container, pull the latest image, and rerun the command
```bash
# stop and remove container
podman stop sds && podman rm sds

# pull latest image
podman image pull public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# run container from latest image (on port 8080)
podman run -d -p 8080:80 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# View the latest release information and other instructions here:
# ECR: https://gallery.ecr.aws/access-ci-org-public-containers/support/standalone-sds
# GitHub: https://github.com/access-ci-org/SDS-Public/releases

```

## Enabling HTTPS/SSL

First make sure port 443 is open on your machine:
```bash
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# Verify firewall settings
firewall-cmd --list-ports
```

Create a `ssl` directory and add your keys into it

```bash
# make .ssl directory
mkdir ssl
# add you ssl certificate (cert.pem) and key (key.pem) to the ssl directory

```

Create a nginx config file
```bash
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

```bash
# stop and remove the container
podman stop sds && podman rm sds

# Give sds sudo permissions
usermod -aG docker sds

# rerun the container with the appropriate port and mounts
# You will need to run podman with sudo permissions
sudo podman run -d -p 443:443 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default" \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest
```

## Custom or No Example Use
Make sure a `software_uses` directory exists in your system and navigate to it
```
mkdir software_uses
cd software_uses
```

Inside of each directory create a file with the software name as the file name
```
# e.g. 'touch python' for instructions for the python software
touch <software_name>
```

**Leave the file empty if you don't want any example usage information to show for that software**

Add any example use information into that file in markdown format
```bash
cat << EOF > python
# Entering python environment
To enter a python environment run `python3` on your terminal

# Existing a python environment
To exit a python environment run `exit()` from within the environment
EOF
```

Stop and re-run the container with the following changes:
 - Mount the new example_uses directory to the container (`-v ./software_uses:/sds/software_uses`)

```bash
# stop and remove the container
podman stop sds && podman rm sds

# Give sds sudo permissions
usermod -aG docker sds

# rerun the container with the appropriate port and mounts
# You will need to run podman with sudo permissions
sudo podman run -d -p 443:443 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml" \
-v ./spider_data:/sds/spider_data \
-v ./websites:/sds/app/data/websites \
-v ./logs:/var/log/supervisor \
./software_uses:/sds/software_uses \
--mount type=bind,source="./logs/sds-internal.log",target="/sds/logs/sds.log" \
--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default" \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest
```




