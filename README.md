# WTF DevOps

[![WTF](https://img.shields.io/badge/wtf-approved-ff69b4.svg)](https://github.com/shipitfast/shipitfast) [![No BS](https://img.shields.io/badge/bullshit-0%25-green.svg)](https://github.com/shipitfast/shipitfast)

> deployment tools that work without requiring kubernetes PhD

## wtf is this?

tired of devops "solutions" that need a team of specialists to deploy hello world?

we only list deployment tools that:
- ✅ **actually ship software** (not just move yaml around)
- ✅ **developers can use** (not just devops wizards)
- ✅ **don't require 6 month learning curve** (looking at you, k8s)
- ✅ **solve real problems** (not create new ones)

## the short list

| name | what it does | why it's here | who uses it |
|------|-------------|---------------|-------------|
| [Docker](https://docker.com) | containerization that works | runs everywhere, zero surprises | google, netflix, everyone |
| [Terraform](https://terraform.io) | infrastructure as code | multi-cloud, declarative, mature | spotify, github, paypal |
| [GitHub Actions](https://github.com/features/actions) | CI/CD without yaml hell | free, integrated, actually works | microsoft, discord, netflix |
| [Traefik](https://traefik.io) | reverse proxy + load balancer | auto SSL, docker-aware, simple config | trivago, blablacar |
| [Ansible](https://ansible.com) | server configuration management | agentless, yaml-based, gets shit done | red hat, nasa |

## honorable mentions

- [Nginx](https://nginx.org) - battle-tested reverse proxy, more manual setup
- [Caddy](https://caddyserver.com) - automatic HTTPS, simpler than nginx
- [GitLab CI](https://gitlab.com/gitlab-org/gitlab-ci) - decent CI/CD, if you use gitlab  
- [Digital Ocean App Platform](https://digitalocean.com/products/app-platform) - heroku-like, good for simple apps
- [Railway](https://railway.app) - git push to deploy, nice for prototypes

## avoid like plague

- **Kubernetes for small teams** - you need 5 devops engineers to run hello world
- **Jenkins** - works but feels like 2005 (which it's from)
- **Complex CI/CD platforms** - if setup takes longer than development, no
- **Microservice orchestrators** - solve problems you create by choosing microservices
- **Enterprise devops platforms** - $50k/month to deploy what heroku does for $25

## devops decision tree

### **simple web app: docker + github actions + vps**
```dockerfile
# Dockerfile that works
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]

# build: docker build -t myapp .
# run: docker run -p 3000:3000 myapp
# deploy: push to registry, pull on server
```

### **infrastructure: terraform**  
```hcl
# main.tf - simple but powerful
provider "digitalocean" {
  token = var.do_token
}

resource "digitalocean_droplet" "web" {
  image  = "ubuntu-20-04-x64"
  name   = "web-server"
  region = "nyc1"
  size   = "s-1vcpu-1gb"
  
  user_data = file("setup.sh")
}

resource "digitalocean_domain" "default" {
  name       = "example.com"
  ip_address = digitalocean_droplet.web.ipv4_address
}

# terraform plan && terraform apply
# infrastructure as code, version controlled
```

### **CI/CD: github actions**
```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t myapp .
      
    - name: Push to registry  
      run: |
        echo ${{ secrets.DOCKER_TOKEN }} | docker login -u ${{ secrets.DOCKER_USER }} --password-stdin
        docker push myapp
        
    - name: Deploy to server
      run: |
        ssh user@server "docker pull myapp && docker restart myapp"

# git push = automatic deployment
```

### **load balancing + SSL: traefik**
```yaml
# docker-compose.yml with traefik
version: '3.8'
services:
  traefik:
    image: traefik:v2.9
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command:
      - --providers.docker=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.email=admin@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      
  app:
    image: myapp:latest
    labels:
      - traefik.http.routers.app.rule=Host(`example.com`)
      - traefik.http.routers.app.tls.certresolver=letsencrypt

# automatic SSL certificates, load balancing, zero downtime deploys
```

### **server configuration: ansible**
```yaml
# playbook.yml - server setup automation  
- hosts: servers
  become: yes
  tasks:
  - name: Install Docker
    apt:
      name: docker.io
      state: present
      
  - name: Start Docker service
    service:
      name: docker
      state: started
      enabled: yes
      
  - name: Install Docker Compose
    pip:
      name: docker-compose
      
  - name: Deploy application
    docker_compose:
      project_src: /opt/myapp
      
# ansible-playbook -i inventory playbook.yml
# reproducible server configuration
```

## real-world deployment patterns

### **single server deployment:**
```markdown
architecture:
- nginx/traefik (reverse proxy)
- docker containers (application)
- postgresql (database)
- redis (cache/sessions)

pros: simple, cheap, easy to debug
cons: single point of failure
good for: mvps, small apps, side projects
```

### **multi-server setup:**
```markdown  
architecture:
- load balancer (traefik/nginx)
- multiple app servers (docker swarm)
- managed database (RDS/managed postgres) 
- managed cache (ElastiCache/managed redis)

pros: redundancy, can handle traffic spikes  
cons: more complex, higher costs
good for: growing companies, B2B apps
```

### **cloud-native:**
```markdown
architecture:
- managed container service (ECS/Cloud Run)
- managed database (RDS/Cloud SQL)
- CDN (CloudFront/CloudFlare)
- monitoring (CloudWatch/Datadog)

pros: scales automatically, less maintenance
cons: vendor lock-in, can get expensive
good for: high-growth companies, variable traffic
```

## devops anti-patterns

### ❌ **premature kubernetes**
```yaml
# don't do this unless you have 10+ services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: hello-world:latest
        
# 47 lines of yaml to run one container
```

### ❌ **microservices without monolith experience**
```markdown
# don't start with this
user-service → authentication-service → profile-service → notification-service

# start with this  
monolith {
  user management
  authentication  
  profiles
  notifications
}

# extract services when you have real scaling problems
```

### ❌ **overengineered CI/CD**
```yaml
# don't build this on day one
stages:
  - lint
  - unit-tests  
  - integration-tests
  - security-scan
  - build
  - deploy-to-staging
  - smoke-tests
  - load-tests  
  - deploy-to-production
  - rollback-monitoring
  
# start with this
stages:
  - test
  - deploy
```

### ❌ **infrastructure snowflakes**
```bash
# don't do manual server setup
ssh production-server
sudo apt install thing
sudo vim /etc/config
sudo systemctl restart service

# use infrastructure as code
terraform apply    # reproducible infrastructure
ansible-playbook setup.yml  # consistent configuration
```

## deployment checklist

### **basic deployment:**
- [ ] containerized application (docker)
- [ ] automated builds (github actions/gitlab ci)
- [ ] infrastructure as code (terraform/pulumi)
- [ ] SSL certificates (let's encrypt/cloudflare)
- [ ] basic monitoring (uptime checks)
- [ ] backup strategy (database backups)

### **production ready:**
- [ ] load balancing (traefik/nginx/cloudflare)
- [ ] database replication (read replicas)
- [ ] zero-downtime deployments (blue/green or rolling)
- [ ] health checks (application and infrastructure)
- [ ] log aggregation (centralized logging)
- [ ] security hardening (firewall, fail2ban, updates)

### **scale-ready:**
- [ ] auto-scaling (horizontal scaling based on metrics)
- [ ] CDN (static asset delivery)
- [ ] caching layers (redis/memcached)
- [ ] database sharding/partitioning
- [ ] disaster recovery (cross-region backups)
- [ ] chaos engineering (test failure scenarios)

## tools by team size

### **solo developer / small team:**
```markdown
hosting: digital ocean droplet / aws lightsail
deployment: docker + github actions  
database: managed postgres (digital ocean / aws rds)
monitoring: uptime robot + sentry
cost: $20-50/month
```

### **growing team (5-20 people):**
```markdown
hosting: aws ecs / google cloud run
deployment: terraform + github actions
database: managed postgres with replicas
monitoring: datadog / new relic
cost: $200-1000/month  
```

### **larger company (20+ people):**
```markdown
hosting: kubernetes (eks/gke) or managed containers
deployment: gitops (flux/argocd) + terraform
database: managed databases + caching layers
monitoring: full observability stack (prometheus/grafana)
cost: $2000+/month
```

## contributing

got devops tools that actually deploy software?

**requirements:**
1. **use it in production** - we don't list tools we haven't operated
2. **show real deployments** - companies using it for real workloads
3. **explain complexity vs benefits** - what problems does it solve?
4. **deployment experience** - how hard is it to set up and maintain?

**red flags:**
- requires dedicated devops team to operate
- solves problems you don't have yet  
- vendor lock-in without clear benefits
- more complex than the application it deploys

## philosophy

**good devops:**
- makes deploying software easier and safer
- reduces manual work and human error
- scales with team and application needs
- integrates with development workflow
- fails fast and provides clear error messages

**bad devops:**
- adds complexity without clear benefits
- requires specialists to understand and maintain
- creates vendor lock-in or technology coupling
- optimizes for theoretical problems vs real needs
- abstracts away important operational details

---

*made by developers who've been paged at 3am because of deployment issues*

**if your deployment process is more complex than your application, you're doing devops wrong**