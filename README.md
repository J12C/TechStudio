# TechStudio

A production-ready Docker setup for WordPress, Prometheus, Grafana & Adminer on Rocky Linux 9.5 — secured, SSL-enabled, and fully containerized.

---

## 📋 Table of Contents

1. [Prerequisites](#-prerequisites)  
2. [Step-by-Step SSH & System Hardening](#-step-by-step-ssh--system-hardening)  
3. [Non-Root User Setup](#-non-root-user-setup)  
4. [Install & Configure Docker](#-install--configure-docker)  
5. [Directory Structure & Volumes](#-directory-structure--volumes)  
6. [Docker Compose Definition](#-docker-compose-definition)  
7. [Prometheus Configuration](#-prometheus-configuration)  
8. [NGINX Reverse-Proxy Config](#-nginx-reverse-proxy-config)  
9. [SSL via Let’s Encrypt](#-ssl-via-lets-encrypt)  
10. [SELinux Adjustments](#-selinux-adjustments)  

---

## 🚧 Prerequisites

- Rocky Linux 9.5 VPS  
- SSH keypair (public/private)  
- Placeholder DNS for your services  

---

## 🔐 Step-by-Step SSH & System Hardening

**1. Connect to the VPS as root**  
```bash
ssh root@<VPS_IP>
```

**2. Change the root password**  
```bash
passwd
```

**3. Install and enable firewalld**  
```bash
dnf install -y firewalld
systemctl enable --now firewalld
```

**4. Allow SSH only from your IP**  
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<YOUR_IP>" port protocol="tcp" port="SSH_PORT" accept'
firewall-cmd --permanent --remove-service=ssh
firewall-cmd --reload
```

**5. Copy your SSH public key to root**  
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@<VPS_IP>
```

**6. Harden SSH configuration**  
```bash
vi /etc/ssh/sshd_config
```
- Set `Port SSH_PORT`
- Set `PermitRootLogin no`
- Set `PubkeyAuthentication yes`
- Set `PasswordAuthentication no`
- Set `MaxAuthTries 3`

```bash
systemctl restart sshd
```

---

## 👤 Non-Root User Setup

**1. Create a non-root user**  
```bash
useradd -m -s /bin/bash usuario
```

**2. Set its password**  
```bash
passwd usuario
```

**3. Add the user to the Docker group**  
```bash
usermod -aG docker usuario
```

---

## 🐳 Install & Configure Docker

**1. Add Docker CE repository**  
```bash
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

**2. Install Docker CE and dependencies**  
```bash
dnf install -y docker-ce docker-ce-cli containerd.io
```

**3. Enable and start Docker**  
```bash
systemctl enable --now docker
```

**4. Switch to the non-root user**  
```bash
su - usuario
```

**5. Verify Docker installation**  
```bash
docker version
```

---

## 📁 Directory Structure & Volumes

**1. Create volume directories**  
```bash
mkdir -p ~/docker_volumes/mysql/data
mkdir -p ~/docker_volumes/wordpress/html
mkdir -p ~/docker_volumes/prometheus/data
touch ~/docker_volumes/prometheus/prometheus.yml
mkdir -p ~/docker_volumes/grafana/data
```

**2. Update system packages**  
```bash
sudo dnf update -y
```

---

## 🏗 Docker Compose Definition

```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    env_file: .env
    volumes:
      - /home/usuario/docker_volumes/mysql/data:/var/lib/mysql
    networks:
      - wp_network

  wordpress:
    image: wordpress:6.8.1-php8.3-apache
    restart: always
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=${WORDPRESS_DB_HOST}
      - WORDPRESS_DB_NAME=${MYSQL_DATABASE}
      - WORDPRESS_DB_USER=${MYSQL_USER}
      - WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}
    depends_on:
      - db
    volumes:
      - /home/usuario/docker_volumes/wordpress/html:/var/www/html
    ports:
      - "81:80"
    networks:
      - wp_network

  prometheus:
    image: prom/prometheus:v3.4.0
    restart: always
    volumes:
      - /home/usuario/docker_volumes/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - /home/usuario/docker_volumes/prometheus/data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:12.0.1
    restart: always
    env_file: .env
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}
    depends_on:
      - prometheus
    volumes:
      - /home/usuario/docker_volumes/grafana/data:/var/lib/grafana
    ports:
      - "3000:3000"

  adminer:
    image: adminer
    restart: always
    ports:
      - "8080:8080"
    networks:
      - wp_network

networks:
  wp_network:
    external: true
```

---

## 📊 Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'docker_metrics'
    metrics_path: /metrics
    static_configs:
      - targets: ['prometheus:9090']
```

---

## 🌐 NGINX Reverse-Proxy Config

```nginx
user nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name <ADMINER_DOMAIN>;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name <ADMINER_DOMAIN>;
        ssl_certificate     /etc/letsencrypt/live/<ADMINER_DOMAIN>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<ADMINER_DOMAIN>/privkey.pem;

        add_header X-Frame-Options DENY always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options nosniff always;
        client_max_body_size 32M;

        location / {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host               $host;
            proxy_set_header X-Real-IP          $remote_addr;
            proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto  $scheme;
            proxy_buffering     on;
            proxy_buffers       4 64k;
            proxy_connect_timeout 30s;
            proxy_read_timeout  30s;
        }
    }

    server {
        listen 443 ssl http2;
        server_name <WORDPRESS_DOMAIN>;
        ssl_certificate     /etc/letsencrypt/live/<WORDPRESS_DOMAIN>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<WORDPRESS_DOMAIN>/privkey.pem;

        add_header X-Frame-Options SAMEORIGIN always;
        add_header X-Content-Type-Options nosniff always;
        client_max_body_size 64M;

        location / {
            proxy_pass http://127.0.0.1:81;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            expires 1h;
        }

        location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|txt)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    server {
        listen 443 ssl http2;
        server_name <GRAFANA_DOMAIN>;
        ssl_certificate     /etc/letsencrypt/live/<GRAFANA_DOMAIN>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<GRAFANA_DOMAIN>/privkey.pem;

        add_header X-Frame-Options SAMEORIGIN always;
        add_header X-Content-Type-Options nosniff always;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host       $host;
            proxy_read_timeout 60s;
        }
    }
}
```

---

## 🔒 SSL via Let’s Encrypt

Certbot uses the Webroot plugin here: it creates a temporary file in your document root for each domain to prove control, then fetches and installs the certificate.

**1. Install Certbot**  
```bash
dnf install -y certbot
```

**2. Obtain a certificate for WordPress**  
```bash
certbot certonly --standalone --register-unsafely-without-email --agree-tos -d <WORDPRESS_DOMAIN>
```

**3. Obtain a certificate for Grafana**  
```bash
certbot certonly --standalone --register-unsafely-without-email --agree-tos -d <GRAFANA_DOMAIN>
```

**4. Obtain a certificate for Adminer**  
```bash
certbot certonly --standalone --register-unsafely-without-email --agree-tos -d <ADMINER_DOMAIN>
```

**5. Reload NGINX to apply new certs**  
```bash
systemctl reload nginx
```

---

## 🔧 SELinux Adjustments

Enable NGINX to bind to the container ports:

```bash
semanage port -a -t http_port_t -p tcp 81
```
```bash
semanage port -a -t http_port_t -p tcp 8080
```
```bash
semanage port -a -t http_port_t -p tcp 3000
```
