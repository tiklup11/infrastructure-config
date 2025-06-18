# Infrastructure Playbook - Complete Guide

## Table of Contents

1. [Adding a New Application](#1-adding-a-new-application)
2. [Deploying to a New VPS](#2-deploying-to-a-new-vps)
3. [Setting Up Dev/Staging/Prod Environments](#3-setting-up-devstaging-prod-environments)
4. [Domain Configuration](#4-domain-configuration)
5. [Adding Backend Services](#5-adding-backend-services)
6. [Database Management](#6-database-management)
7. [SSL/HTTPS Setup](#7-sslhttps-setup)
8. [Monitoring and Logging](#8-monitoring-and-logging)
9. [Backup and Recovery](#9-backup-and-recovery)
10. [Scaling Applications](#10-scaling-applications)
11. [Troubleshooting Guide](#11-troubleshooting-guide)
12. [Security Best Practices](#12-security-best-practices)

---

## 1. Adding a New Application

### 1.1 Frontend Application (React/Vue/etc)

#### Step 1: Create New App Repository

```
new-webapp/
├── .github/workflows/deploy.yml
├── Dockerfile
├── nginx.conf
├── package.json
└── src/
```

#### Step 2: Create deploy.yml for the app

```yaml
name: Build and Deploy New Web App

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/new-webapp:latest
            ghcr.io/${{ github.repository_owner }}/new-webapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd ~/infrastructure
            docker pull ghcr.io/${{ github.repository_owner }}/new-webapp:latest
            docker-compose stop new-webapp || true
            docker-compose rm -f new-webapp || true
            export NEW_WEBAPP_VERSION=${{ github.sha }}
            export GITHUB_USERNAME=${{ github.repository_owner }}
            export VPS_IP=${{ secrets.VPS_HOST }}
            docker-compose up -d new-webapp --no-deps
```

#### Step 3: Add to infrastructure docker-compose.yml

```yaml
new-webapp:
  image: ghcr.io/${GITHUB_USERNAME}/new-webapp:${NEW_WEBAPP_VERSION:-latest}
  container_name: new-webapp
  restart: unless-stopped
  labels:
    - "traefik.enable=true"
    - "traefik.docker.network=web"
    - "traefik.http.routers.newapp.rule=Host(`${VPS_IP}`) && PathPrefix(`/newapp`)"
    - "traefik.http.routers.newapp.entrypoints=web"
    - "traefik.http.services.newapp.loadbalancer.server.port=80"
    - "traefik.http.middlewares.newapp-stripprefix.stripprefix.prefixes=/newapp"
    - "traefik.http.routers.newapp.middlewares=newapp-stripprefix"
  networks:
    - web
  depends_on:
    - traefik
```

### 1.2 Backend Application (Node.js/Python/Go)

#### Additional Dockerfile for Backend

```dockerfile
# Node.js example
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

#### Docker-compose entry for backend

```yaml
api-service:
  image: ghcr.io/${GITHUB_USERNAME}/api-service:${API_VERSION:-latest}
  container_name: api-service
  restart: unless-stopped
  environment:
    - NODE_ENV=production
    - DATABASE_URL=${DATABASE_URL}
    - JWT_SECRET=${JWT_SECRET}
  labels:
    - "traefik.enable=true"
    - "traefik.docker.network=web"
    - "traefik.http.routers.api.rule=Host(`${VPS_IP}`) && PathPrefix(`/api`)"
    - "traefik.http.routers.api.entrypoints=web"
    - "traefik.http.services.api.loadbalancer.server.port=3000"
  networks:
    - web
  depends_on:
    - traefik
    - postgres
```

---

## 2. Deploying to a New VPS

### 2.1 Multi-VPS Setup

#### Option A: Separate Infrastructure Repos

```
infrastructure-prod/     # Production VPS
infrastructure-staging/  # Staging VPS
infrastructure-dev/      # Development VPS
```

#### Option B: Single Repo with Multiple Configs

```
infrastructure/
├── environments/
│   ├── prod/
│   │   ├── docker-compose.yml
│   │   └── .env
│   ├── staging/
│   │   ├── docker-compose.yml
│   │   └── .env
│   └── dev/
│       ├── docker-compose.yml
│       └── .env
└── .github/workflows/
    ├── deploy-prod.yml
    ├── deploy-staging.yml
    └── deploy-dev.yml
```

### 2.2 New VPS Setup Checklist

```bash
# 1. Initial server setup
ssh root@new-vps-ip
adduser deploy
usermod -aG sudo deploy
su - deploy

# 2. Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 3. Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 4. Setup firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp  # Traefik dashboard
sudo ufw enable

# 5. Create infrastructure directory
mkdir ~/infrastructure
cd ~/infrastructure

# 6. Generate SSH key for GitHub Actions
ssh-keygen -t ed25519 -C "github-actions"
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_ed25519  # Copy this for GitHub secrets
```

### 2.3 GitHub Secrets for New VPS

Add to each app repository:

- `PROD_VPS_HOST`: Production VPS IP
- `PROD_VPS_USERNAME`: deploy
- `PROD_VPS_SSH_KEY`: Private key from step 6
- `STAGING_VPS_HOST`: Staging VPS IP
- `STAGING_VPS_USERNAME`: deploy
- `STAGING_VPS_SSH_KEY`: Private key

---

## 3. Setting Up Dev/Staging/Prod Environments

### 3.1 Environment-Specific Workflows

#### deploy-prod.yml

```yaml
name: Deploy to Production

on:
  push:
    tags:
      - "v*" # Only deploy tags like v1.0.0

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production # Requires approval
    steps:
      # ... deployment steps using PROD secrets
```

#### deploy-staging.yml

```yaml
name: Deploy to Staging

on:
  push:
    branches: [staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      # ... deployment steps using STAGING secrets
```

### 3.2 Environment Variables

#### .env.prod

```env
ENVIRONMENT=production
DOMAIN=yourdomain.com
API_URL=https://api.yourdomain.com
ENABLE_DEBUG=false
LOG_LEVEL=error
```

#### .env.staging

```env
ENVIRONMENT=staging
DOMAIN=staging.yourdomain.com
API_URL=https://api-staging.yourdomain.com
ENABLE_DEBUG=true
LOG_LEVEL=debug
```

### 3.3 Docker Compose Override

#### docker-compose.prod.yml

```yaml
services:
  webapp:
    environment:
      - REACT_APP_API_URL=https://api.yourdomain.com
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
```

Use with: `docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d`

---

## 4. Domain Configuration

### 4.1 DNS Setup

Add these records at your domain registrar:

```
A     @        129.154.247.216  # Root domain
A     www      129.154.247.216  # www subdomain
A     api      129.154.247.216  # API subdomain
A     staging  STAGING_VPS_IP   # Staging subdomain
```

### 4.2 Update Traefik Rules

#### For root domain (pretty-little-gift)

```yaml
labels:
  - "traefik.http.routers.pretty.rule=Host(`yourdomain.com`, `www.yourdomain.com`)"
  - "traefik.http.routers.pretty.entrypoints=web,websecure"
  - "traefik.http.routers.pretty.tls=true"
  - "traefik.http.routers.pretty.tls.certresolver=letsencrypt"
```

#### For subdomains

```yaml
labels:
  - "traefik.http.routers.api.rule=Host(`api.yourdomain.com`)"
  - "traefik.http.routers.people.rule=Host(`people.yourdomain.com`)"
```

### 4.3 Enable HTTPS with Let's Encrypt

#### Update traefik.yml

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com
      storage: acme.json
      httpChallenge:
        entryPoint: web
```

#### Create acme.json

```bash
touch acme.json
chmod 600 acme.json
```

---

## 5. Adding Backend Services

### 5.1 Database Setup

#### PostgreSQL

```yaml
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  backend:
    driver: bridge
```

#### MongoDB

```yaml
  mongodb:
    image: mongo:6
    container_name: mongodb
    restart: unless-stopped
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_USER}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    volumes:
      - mongo_data:/data/db
    networks:
      - backend

volumes:
  mongo_data:
```

### 5.2 Redis Cache

```yaml
redis:
  image: redis:7-alpine
  container_name: redis
  restart: unless-stopped
  command: redis-server --requirepass ${REDIS_PASSWORD}
  networks:
    - backend
```

### 5.3 Message Queue (RabbitMQ)

```yaml
rabbitmq:
  image: rabbitmq:3-management-alpine
  container_name: rabbitmq
  restart: unless-stopped
  environment:
    - RABBITMQ_DEFAULT_USER=${RABBITMQ_USER}
    - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
  ports:
    - "15672:15672" # Management UI
  networks:
    - backend
```

---

## 6. Database Management

### 6.1 Backup Strategy

#### Automated PostgreSQL Backup

```yaml
postgres-backup:
  image: prodrigestivill/postgres-backup-local
  container_name: postgres-backup
  restart: unless-stopped
  environment:
    - POSTGRES_HOST=postgres
    - POSTGRES_DB=${DB_NAME}
    - POSTGRES_USER=${DB_USER}
    - POSTGRES_PASSWORD=${DB_PASSWORD}
    - SCHEDULE=@daily
    - BACKUP_KEEP_DAYS=7
  volumes:
    - ./backups:/backups
  depends_on:
    - postgres
```

#### Manual Backup Commands

```bash
# Backup
docker exec postgres pg_dump -U user dbname > backup_$(date +%Y%m%d).sql

# Restore
docker exec -i postgres psql -U user dbname < backup_20230101.sql
```

### 6.2 Database Migrations

#### Using Flyway

```yaml
flyway:
  image: flyway/flyway
  command: migrate
  volumes:
    - ./migrations:/flyway/sql
  environment:
    - FLYWAY_URL=jdbc:postgresql://postgres:5432/${DB_NAME}
    - FLYWAY_USER=${DB_USER}
    - FLYWAY_PASSWORD=${DB_PASSWORD}
  depends_on:
    - postgres
```

---

## 7. SSL/HTTPS Setup

### 7.1 Wildcard Certificate

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

### 7.2 Certificate for Multiple Domains

```yaml
labels:
  - "traefik.http.routers.app.tls.domains[0].main=yourdomain.com"
  - "traefik.http.routers.app.tls.domains[0].sans=*.yourdomain.com"
```

---

## 8. Monitoring and Logging

### 8.1 Prometheus + Grafana

```yaml
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`monitor.yourdomain.com`)"
    networks:
      - web
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

### 8.2 Log Aggregation

```yaml
loki:
  image: grafana/loki
  container_name: loki
  command: -config.file=/etc/loki/local-config.yaml
  networks:
    - monitoring

promtail:
  image: grafana/promtail
  container_name: promtail
  volumes:
    - /var/log:/var/log
    - ./promtail-config.yml:/etc/promtail/config.yml
  command: -config.file=/etc/promtail/config.yml
  networks:
    - monitoring
```

---

## 9. Backup and Recovery

### 9.1 Full System Backup

```bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/home/deploy/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup Docker volumes
docker run --rm -v $(pwd):/backup -v /var/lib/docker/volumes:/volumes alpine tar czf /backup/volumes.tar.gz /volumes

# Backup infrastructure config
tar czf $BACKUP_DIR/infrastructure.tar.gz ~/infrastructure

# Backup databases
docker exec postgres pg_dumpall -U postgres > $BACKUP_DIR/postgres_all.sql
docker exec mongodb mongodump --archive=$BACKUP_DIR/mongodb.archive

# Upload to S3 or external storage
aws s3 sync $BACKUP_DIR s3://your-backup-bucket/$(date +%Y%m%d)/
```

### 9.2 Disaster Recovery Plan

1. **Infrastructure as Code**: All configs in Git
2. **Database Backups**: Daily automated backups
3. **Image Registry**: All images in GitHub Container Registry
4. **Documentation**: This playbook + runbooks
5. **Recovery Time Objective (RTO)**: < 1 hour
6. **Recovery Point Objective (RPO)**: < 24 hours

---

## 10. Scaling Applications

### 10.1 Horizontal Scaling with Docker Swarm

```bash
# Initialize swarm
docker swarm init --advertise-addr VPS_IP

# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# Scale service
docker service scale myapp_webapp=3
```

### 10.2 Load Balancing

```yaml
webapp:
  image: ghcr.io/${GITHUB_USERNAME}/webapp:latest
  deploy:
    replicas: 3
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.webapp.loadbalancer.sticky.cookie=true"
```

### 10.3 Using External Load Balancer

- Oracle Load Balancer
- Cloudflare Load Balancing
- AWS Application Load Balancer

---

## 11. Troubleshooting Guide

### 11.1 Common Issues

#### Container Won't Start

```bash
# Check logs
docker logs container_name
docker-compose logs service_name

# Check events
docker events --since 10m

# Inspect container
docker inspect container_name
```

#### Port Already in Use

```bash
# Find process using port
sudo lsof -i :80
sudo netstat -tulpn | grep :80

# Kill process
sudo kill -9 PID
```

#### Disk Space Issues

```bash
# Check disk usage
df -h
docker system df

# Clean up
docker system prune -a --volumes
docker image prune -a
docker volume prune
```

### 11.2 Performance Issues

#### Resource Usage

```bash
# Check container stats
docker stats

# Limit resources in docker-compose
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 512M
    reservations:
      cpus: '0.25'
      memory: 256M
```

---

## 12. Security Best Practices

### 12.1 Container Security

```yaml
webapp:
  image: webapp:latest
  security_opt:
    - no-new-privileges:true
  read_only: true
  user: "1000:1000" # Non-root user
  cap_drop:
    - ALL
  cap_add:
    - NET_BIND_SERVICE
```

### 12.2 Secrets Management

```yaml
# Use Docker secrets
secrets:
  db_password:
    external: true
  api_key:
    external: true

services:
  app:
    secrets:
      - db_password
      - api_key
```

### 12.3 Network Security

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true # No external access
```

### 12.4 Regular Updates

```bash
# Update all images
docker-compose pull
docker-compose up -d

# Security scanning
docker scan webapp:latest
```

---

## Quick Reference Commands

```bash
# Deploy new app
docker-compose up -d new-app --no-deps

# View logs
docker-compose logs -f app-name

# Restart service
docker-compose restart app-name

# Scale service
docker-compose up -d --scale webapp=3

# Execute command in container
docker-compose exec app-name sh

# Update single service
docker-compose pull app-name
docker-compose up -d app-name --no-deps

# Full system update
docker-compose pull
docker-compose up -d

# Emergency rollback
docker pull ghcr.io/user/app:previous-sha
docker-compose up -d app --no-deps
```

---

## Future Considerations

1. **Kubernetes Migration**: When you outgrow Docker Compose
2. **CI/CD Pipeline**: Jenkins, GitLab CI, or GitHub Actions improvements
3. **Service Mesh**: Istio or Linkerd for microservices
4. **Observability**: Datadog, New Relic, or ELK stack
5. **Multi-Region**: Deploy across multiple data centers
6. **CDN Integration**: Cloudflare, Fastly for static assets
7. **API Gateway**: Kong, Tyk for API management
8. **Secrets Rotation**: HashiCorp Vault integration
9. **Compliance**: GDPR, HIPAA, SOC2 considerations
10. **Cost Optimization**: Resource monitoring and rightsizing

---

## Conclusion

This playbook covers most scenarios you'll encounter. Key principles:

- **Everything as Code**: All configurations in Git
- **Immutable Infrastructure**: Never modify running containers
- **Automation First**: Minimize manual operations
- **Security by Default**: Follow least privilege principle
- **Monitor Everything**: You can't improve what you don't measure

Remember: Start simple, iterate, and scale as needed. Don't over-engineer from day one!
