# VM Setup, Docker, CI/CD, Nginx & SSL Deployment Guide

This document explains how to configure a Linux VM, connect it securely to GitHub, deploy an application using **Docker** with a **self-hosted GitHub runner**, and expose it using **Nginx with SSL**.

---

## ⚠️ Important Notes (Read First)

* If you **already have SSH access configured with a non-root user**,
  👉 **Skip Section 1**.
* If your VM is **already connected to GitHub via SSH** and
  `ssh -T git@github.com` works,
  👉 **Skip Section 4**.

---

## Prerequisites

* Ubuntu 20.04 / 22.04 VM
* Domain name pointed to VM public IP
* GitHub repository (SSH access enabled)
* Open ports: **22, 80, 443**

---

## 1. VM Access via SSH (Non-Root User)

> Skip this section if SSH access is already configured.

### Create a non-root user

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

### Secure SSH configuration (recommended)

```bash
sudo nano /etc/ssh/sshd_config
```

Update or confirm:

```text
PermitRootLogin no
PasswordAuthentication no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

Login:

```bash
ssh deploy@VM_IP
```

---

## 2. Install Docker (If Not Installed)

Check Docker:

```bash
docker --version
```

Install Docker if missing:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 3. Give Docker Access to SSH User

```bash
sudo usermod -aG docker deploy
newgrp docker
```

Verify:

```bash
docker ps
```

---

## 4. Connect VM with GitHub via SSH

> Skip this section if GitHub SSH connection already works.

### Generate SSH key on VM

```bash
ssh-keygen -t ed25519 -C "deploy@vm"
```

Press **Enter** for defaults and empty passphrase.

---

### Add SSH key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub
```

Add the key in:
**GitHub → Settings → SSH and GPG Keys → New SSH Key**

---

### Test connection

```bash
ssh -T git@github.com
```

Expected:

```text
Hi <username>! You've successfully authenticated.
```

---

## 5. Create Repository Directory & Set Permissions

### Create directory

```bash
sudo mkdir -p /home/deploy/apps/project_name
```

### Set correct ownership (IMPORTANT)

```bash
sudo chown -R deploy:deploy /home/deploy/apps/project_name
```

Verify:

```bash
ls -ld /home/deploy/apps/project_name
```

Expected:

```text
deploy deploy /home/deploy/apps/project_name
```

---

## 6. Clone Repository via SSH

```bash
git clone -b "branch_name" "REPO_URL" "REPO_DIR"
```

---

## 7. Add GitHub Self-Hosted Runner (Persistent)

- Go to your GitHub repository or organization → **Settings → Actions → Runners** → **Add runner** (if it does not exist).
- Copy and run the commands GitHub provides to **download and configure the runner** on your VM.

### Run runner as a service (MANDATORY)

```bash
sudo ./svc.sh install
sudo ./svc.sh start
```

✅ Runner stays alive after terminal closes or VM restarts.

---

## 8. Dockerfile (Required)

Your project **must include a Dockerfile**.

### Example (Vite App)

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 4173
CMD ["npm", "run", "preview"]
```

---

## 9. GitHub Workflow (CI/CD)

Create:

```text
.github/workflows/deploy.yml
```

### Example Workflow

```yaml
name: Build and Deploy Vite App (Self-Hosted)

on:
  push:
    branches:
      - branch_name

jobs:
  build-and-deploy:
    runs-on: self-hosted
    env:
      REPO_URL: git@github.com:${{ github.repository }}.git
      REPO_DIR: /home/deploy/apps/project_name
      BRANCH: branch_name

    steps:
      - name: Checkout workflow repo
        uses: actions/checkout@v3

      - name: Add repo to safe directories
        run: |
          git config --global --add safe.directory "${{ env.REPO_DIR }}"

      - name: Clone or pull latest code
        run: |
          if [ -d "${{ env.REPO_DIR }}/.git" ]; then
            cd "${{ env.REPO_DIR }}"
            git fetch origin ${{ env.BRANCH }}
            git checkout ${{ env.BRANCH }}
            git reset --hard origin/${{ env.BRANCH }}
          else
            git clone -b ${{ env.BRANCH }} ${{ env.REPO_URL }} ${{ env.REPO_DIR }}
          fi

      - name: Build Docker image
        run: |
          cd "${{ env.REPO_DIR }}"
          docker build -t image_name .

      - name: Stop old container
        run: |
          docker stop container_name || true
          docker rm container_name || true

      - name: Run container
        run: |
          docker run -d \
          -p vm_port:docker_image_port \
          --name container_name image_name
```

---

## 10. Create Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/project_name
```

```nginx
server {
    server_name your_domain;

    location / {
        proxy_connect_timeout 10m;
        proxy_send_timeout    10m;
        proxy_read_timeout    10m;
        send_timeout          10m;

        proxy_pass http://127.0.0.1:your_port;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 11. Enable Nginx Site

```bash
sudo ln -s /etc/nginx/sites-available/project_name /etc/nginx/sites-enabled/
```

---

## 12. Check Nginx Syntax

```bash
sudo nginx -t
```

---

## 13. Reload Nginx

```bash
sudo systemctl reload nginx
```

---

## 14. Install SSL Using Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your_domain
```

Enable auto-renew:

```bash
sudo systemctl enable certbot.timer
```

---

## 15. Verify Deployment

Open in browser:

```text
https://your_domain
```

✅ Application should be live with HTTPS enabled.

---

## Best Practices

* Never deploy as `root`
* Always run GitHub runner as a **service**
* Use environment variables for secrets
* Monitor logs (`docker logs`, `/var/log/nginx`)
* Keep Docker images small


## 🔄 Optional: Zero-Downtime Deployment (Recommended for Production)

> ⚠️ **Optional Section**
> If brief downtime during deployment is acceptable, you may skip this section.
> For **production environments**, zero-downtime deployment is **strongly recommended**.

This setup uses a **Blue-Green deployment strategy** with **Docker + Nginx**, ensuring users never experience service interruption during deployments.

---

## How Zero-Downtime Deployment Works

* Two containers run simultaneously:

  * **Blue** → current live version
  * **Green** → new version
* Nginx switches traffic instantly between them
* Old container is stopped **after** the switch

Users remain connected without dropped requests.

---

## Architecture Overview

```text
User
 ↓
Nginx (80 / 443)
 ↓
localhost:3001 → app-blue (live)
localhost:3002 → app-green (new)
```

---

## Port Strategy

| Container | Port |
| --------- | ---- |
| Blue      | 3001 |
| Green     | 3002 |

---

## Nginx Upstream Configuration

Update your Nginx config to use an upstream block:

```nginx
upstream app_backend {
    server 127.0.0.1:3001; # active container
}

server {
    server_name your_domain;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Reload Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## Zero-Downtime Deployment Flow

1. Build new Docker image
2. Start **inactive container** (Blue or Green)
3. Wait for application to become healthy
4. Switch Nginx upstream port
5. Reload Nginx (no connection drop)
6. Stop and remove old container

---

## GitHub Actions – Zero-Downtime Deploy Step

Replace the **stop & run container** steps with the following:

```yaml
- name: Zero Downtime Deploy
  run: |
    NGINX_CONF="/etc/nginx/sites-enabled/project_name"
    ACTIVE_PORT=$(grep -oP 'server 127.0.0.1:\K[0-9]+' $NGINX_CONF)

    if [ "$ACTIVE_PORT" = "3001" ]; then
      NEW_PORT=3002
      NEW_NAME=app-green
      OLD_NAME=app-blue
    else
      NEW_PORT=3001
      NEW_NAME=app-blue
      OLD_NAME=app-green
    fi

    docker run -d \
      --name $NEW_NAME \
      -p $NEW_PORT:docker_image_port \
      image_name

    echo "Waiting for container to be ready..."
    sleep 10

    sed -i "s/server 127.0.0.1:$ACTIVE_PORT;/server 127.0.0.1:$NEW_PORT;/" $NGINX_CONF

    sudo nginx -t && sudo systemctl reload nginx

    docker stop $OLD_NAME
    docker rm $OLD_NAME
```

---

## Rollback Strategy (Instant)

If a deployment fails or behaves incorrectly:

```bash
sed -i 's/3002/3001/' /etc/nginx/sites-enabled/project_name
sudo systemctl reload nginx
```

⏱ Rollback time: **< 1 second**

---

## Benefits

* ✅ Zero downtime for users
* ✅ Instant rollback
* ✅ No extra infrastructure
* ✅ Works with existing Docker + Nginx setup

---

## When to Use This

| Environment | Recommendation           |
| ----------- | ------------------------ |
| Development | Optional                 |
| Staging     | Recommended              |
| Production  | **Strongly recommended** |

---

## Notes

* Application should be **stateless**
* Use external DB / cache
* Add health checks for extra safety
* Ensure GitHub runner user has permission to reload Nginx


