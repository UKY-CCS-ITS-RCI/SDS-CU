# SDS Deployment Guide @ CU

This guide explains how to set up and run **SDS** on a Linux system using Podman.
Follow each step in order — starting with Podman installation, then user and firewall setup, preparing the data directory, and running the container.
For the bare minimum setup see https://github.com/access-ci-org/SDS-Public/tree/stand-alone
For more detailed customization see https://github.com/access-ci-org/SDS-Public/blob/stand-alone/SDS_SETUP.md

## Table of Contents

1. [Step 0 : Ensure Podman Is Installed](#step-0--ensure-podman-is-installed)
2. [Step 1 : Create User 'sds' and Configure Podman Access](#step-1--create-user-sds-and-configure-podman-access)
3. [Step 2 : Pull Image](#step-2--pull-image)
4. [Step 3 : Prepare the Data Directory](#step-3--prepare-the-data-directory)
5. [Step 4 : Run the Container](#step-4--run-the-container)
6. [Step 5 : Stopping and Deleting Containers](#step-5--stopping-and-deleting-containers)
7. [Updating SDS](#updating-sds)
8. [Enabling HTTPS/SSL](#enabling-httpsssl)
9. [Custom or No Example Use](#custom-or-no-example-use)

---

## Step 0 : Ensure Podman is installed
This should already be done on your system image

---

## Step 1 : Create user 'sds' and configure Podman access

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

## Step 2 : Pull image

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

## Step 3 : Prepare the data directory

SDS reads all inputs from a single `data/` directory next to your `config.yaml`,
and writes its own state (database, logs, analytics, cached website titles)
under `data/state/` — no other directories or log files need to be created.

```bash
# Create resource directories and download module files
mkdir -p data/spider_data/alpine data/spider_data/blanca

wget -O data/spider_data/alpine/alpine_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/alpine_modulespider.out
wget -O data/spider_data/blanca/blanca_modulespider.out https://raw.githubusercontent.com/UKY-CCS-ITS-RCI/SDS-CU/main/blanca_modulespider.out
```

---

## Step 4 : Run the container

The container expects exactly one data mount at `/sds/data` (plus the config file):

```bash
podman run -d -p 8080:80 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml",relabel=private \
-v ./data:/sds/data:Z \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# Verify running containers
podman ps -a
```

Notes:
 - The container runs as an unprivileged user (UID 1000) and takes ownership of
   `./data` at startup, so everything SDS writes is owned by UID 1000 rather than
   root. The `:Z` on the data volume handles the SELinux labeling.
 - If `./data` is on NFS with `root_squash`, the startup ownership fixup is
   skipped — make sure the export allows UID 1000 to write.

---

## Step 5 : Stopping and deleting containers

**SDS automatically picks up changes to files under `./data/` — just edit them on the host, no container restart needed. Only changes to `config.yaml` (or adding a new mount) require stopping and re-running the container.**

Logs are written to `./data/state/logs/` on the host (`sds.log`, `sds-stdout.log`, `nginx-access.log`, `nginx-error.log`, `supervisord.log`).

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

**That's it!**
Your SDS service should now be running and accessible on port **8080**.

---

## Updating SDS

### One-time migration from pre-v1.4.0

If your existing deployment uses the old multi-mount layout (separate
`spider_data`, `websites`, `logs`, ... mounts), run the migration script once
before updating — the v1.4.0+ container refuses to start on the old layout:

```bash
# stop and remove the old container
podman stop sds && podman rm sds

# move existing files into the new ./data/ layout
wget https://raw.githubusercontent.com/access-ci-org/SDS-Public/stand-alone/migrate_data_layout.sh
bash migrate_data_layout.sh
```

The script moves your existing files into `./data/` without overwriting
anything and prints the new run command (shown as `docker run` — substitute
`podman` for `docker`, the flags are identical). After it finishes, continue
below.

### Regular update

Stop and remove the current container, pull the latest image, and rerun the command

```bash
# stop and remove container
podman stop sds && podman rm sds

# pull latest image
podman image pull public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest

# run container from latest image (on port 8080)
podman run -d -p 8080:80 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml",relabel=private \
-v ./data:/sds/data:Z \
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
# make ssl directory
mkdir ssl
# add your ssl certificate (cert.pem) and key (key.pem) to the ssl directory

```

Create a nginx config file

```bash
cat << 'EOF' > nginx.conf
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    access_log /sds/data/state/logs/nginx-access.log;

    client_max_body_size 100M;
}
EOF
```

Stop and re-run the container with the following changes:
 - Add the appropriate ports (change -p 8080:80 to -p 443:443)
 - Mount the new config to the container (--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default")
 - Mount the ssl directory (`-v ./ssl:/etc/nginx/ssl:ro,Z` the `Z` is for selinux)
```bash
# stop and remove the container
podman stop sds && podman rm sds

# rerun the container with the appropriate port and mounts
# binding port 443 requires root, so run podman with sudo
sudo podman run -d -p 443:443 \
--mount type=bind,source="./config.yaml",target="/sds/config.yaml",relabel=private \
-v ./data:/sds/data:Z \
--mount type=bind,source="./nginx.conf",target="/etc/nginx/sites-available/default",relabel=private \
-v ./ssl:/etc/nginx/ssl:ro,Z \
--name sds \
public.ecr.aws/access-ci-org-public-containers/support/standalone-sds:latest
```

## Custom or No Example Use

Example use files live inside the data directory, so they reach the container
through the existing data mount — no extra mount or container restart needed.

Make sure a `data/software_uses` directory exists and navigate to it

```
mkdir -p data/software_uses
cd data/software_uses
```

Inside the directory create a file with the software name as the file name
(a `.md` suffix is optional)

```
# e.g. 'touch python' for instructions for the python software
touch <software_name>
```

**Leave the file empty if you don't want any example usage information to show for that software**

Add any example use information into that file in markdown format

```bash
cat << 'EOF' > python
# Entering python environment
To enter a python environment run `python3` on your terminal

# Exiting a python environment
To exit a python environment run `exit()` from within the environment
EOF
```
