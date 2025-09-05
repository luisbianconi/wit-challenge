# Linux Laboratory: Docker-Based Infrastructure for Spring Boot Application

**Author:** Luis Fernando Bianconi  
**Date:** September 2025

---

## 🚀 Objective

This lab teaches how to install, configure, and test a Docker-based infrastructure that exposes a Java Spring Boot application (`wit-cicd-challenge.jar`) through a layered architecture:

- **Load Balancer:** [Traefik](https://traefik.io/)
- **Proxy:** [Nginx](https://nginx.org/)
- **App:** Spring Boot

The goal is to expose the application on the hostname `demowit.local`, accessible from the VirtualBox host.

---

## 📁 Project Structure

```text
/home/wit/demowit
├── app
│   ├── Dockerfile
│   └── wit-cicd-challenge.jar
├── lb
│   ├── traefik.yml
│   ├── dynamic.yml
│   └── certs/
│       ├── demowit.crt
│       └── demowit.key
├── proxy
│   └── nginx.conf
├── docker-compose.yml
└── startup.sh
```

---

## 🛠️ Step-by-Step Installation

### 1. Create User and Install Docker

```bash
sudo useradd -m -s /bin/bash wit
sudo passwd wit
sudo usermod -aG wheel wit
sudo curl get.docker.com | sh
docker --version
sudo systemctl enable --now docker
sudo usermod -aG docker wit
```
> **Logout and login as `wit` to apply Docker group access.**

---

### 2. Spring Boot Application Image

**Path:** `~/demowit/app/Dockerfile`

```dockerfile
FROM cgr.dev/chainguard/jre:latest
WORKDIR /app
COPY wit-cicd-challenge.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```
> The JAR runs on port 8080 and responds on root path (`/`).

---

### 3. Nginx Proxy Configuration

**Path:** `~/demowit/proxy/nginx.conf`

```nginx
worker_processes  1;
events { worker_connections 1024; }

http {
  server {
    listen 8080;

    location /wit-test/ {
      proxy_pass http://app:8080/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /wit-test {
      return 301 /wit-test/;
    }
  }
}
```

---

### 4. Traefik Load Balancer

#### 4.1 Static Configuration

**Path:** `~/demowit/lb/traefik.yml`

```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
  file:
    filename: /etc/traefik/dynamic/dynamic.yml
    watch: true

api:
  dashboard: false
```

#### 4.2 Dynamic Configuration

**Path:** `~/demowit/lb/dynamic.yml`

```yaml
http:
  routers:
    demowit:
      rule: "Host(`demowit.local`)"
      entryPoints: [web]
      service: proxy-svc

    demowit-https:
      rule: "Host(`demowit.local`)"
      entryPoints: [websecure]
      service: proxy-svc
      tls: {}

  services:
    proxy-svc:
      loadBalancer:
        servers:
          - url: "http://proxy:8080"

tls:
  certificates:
    - certFile: /etc/traefik/certs/demowit.crt
      keyFile:  /etc/traefik/certs/demowit.key
```

#### 4.3 SSL Certificates

```bash
mkdir -p ~/demowit/lb/certs
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout ~/demowit/lb/certs/demowit.key \
  -out    ~/demowit/lb/certs/demowit.crt \
  -subj "/CN=demowit.local"
```

---

### 5. Docker Compose

**Path:** `~/demowit/docker-compose.yml`

```yaml
networks:
  demowit-net:
    driver: bridge

services:
  lb:
    image: traefik:v3.0
    container_name: lb
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./lb/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./lb/dynamic.yml:/etc/traefik/dynamic/dynamic.yml:ro
      - ./lb/certs:/etc/traefik/certs:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - demowit-net

  proxy:
    image: nginx:alpine
    container_name: proxy
    volumes:
      - ./proxy/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - demowit-net

  app:
    build:
      context: ./app
    container_name: app
    networks:
      - demowit-net
    environment:
      - JAVA_OPTS=""
```

---

### 6. Startup Script (Optional)

**Path:** `~/demowit/startup.sh`

> Create a script to start the environment and print usage instructions.

---

## ▶️ Execution

```bash
cd ~/demowit
./startup.sh
```

Check running containers:

```bash
docker ps
```

Test the application:

```bash
curl http://demowit.local/wit-test/
# or for HTTPS:
curl -k https://demowit.local/wit-test/
```

**Expected output:**
```
Greetings from WIT!
```

---

## 🏅 Bonus Points

- ✅ SSL with self-signed certificate (Traefik)
- ✅ Docker Volumes for config mounts
- ✅ Docker Compose
- ✅ Lightweight and secure base images (chainguard)
- ✅ /wit-test endpoint rewrite via Nginx

---

## 📌 Conclusion

This lab sets up a production-like 3-tier architecture using Docker containers, Traefik as edge load balancer, Nginx as reverse proxy, and a lightweight Spring Boot application, exposed securely via hostname resolution. This setup is portable, reproducible, and aligned with container best practices.

---
