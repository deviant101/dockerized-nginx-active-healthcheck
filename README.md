# NGINX Active Health Check - Dockerized Solution

A free, open-source NGINX load balancer with **active health checks** using Docker via `nginx_upstream_check_module`. No NGINX Plus required!

## Features

- ✅ **Active Health Checks** - Proactive monitoring of backend servers
- ✅ **Dockerized** - Easy deployment and portability
- ✅ **100% Free** - No NGINX Plus required
- ✅ **Production Ready** - Built from source with optimization
- ✅ **Real-time Status** - Visual health check dashboard

## 🎯 What You'll Build

```
                    ┌─────────────────────┐
                    │   Load Balancer     │
                    │  NGINX (Docker)     │
                    │  Active Health Check│
                    └──────────┬──────────┘
                               │
                ┏━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━┓
                ▼                             ▼
      ┌──────────────────┐          ┌──────────────────┐
      │ Backend Server 1 │          │ Backend Server 2 │
      │ NGINX            │          │ NGINX            │
      │ /health          │          │ /health          │
      └──────────────────┘          └──────────────────┘
```

---

## 📋 Prerequisites

- Docker and Docker Compose installed
- 2+ backend servers with NGINX or any web server
- Backend servers must have a `/health` endpoint

---

## 🚀 Part 1: Build the Load Balancer Image

### Step 1: Clone or Download Files

You need these files:
- `Dockerfile` - Builds NGINX with health check module
- `nginx.conf` - Load balancer configuration
- `docker-compose.yml` - Container orchestration
- `backend-nginx.conf` - Ready-to-use backend server config (optional)

### Step 2: Configure Backend IPs

Edit `nginx.conf` and replace the placeholder IPs with your actual backend server IPs:

```nginx
upstream backend_servers {
    server 192.168.1.10:80 weight=1;  # Replace with Backend Server 1 IP
    server 192.168.1.11:80 weight=1;  # Replace with Backend Server 2 IP
    
    # Health check configuration
    check interval=3000 rise=2 fall=3 timeout=2000 type=http;
    check_http_send "HEAD /health HTTP/1.1\r\nHost: health-check\r\nConnection: close\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

**Important**: Use the actual IP addresses your load balancer can reach (private IPs if on the same network).

### Step 3: Build the Docker Image

```bash
docker build -t nginx-health-check:latest .
```

This will:
- Download NGINX source code
- Download the health check module
- Compile NGINX with the module
- Create a production-ready Docker image

Build time: ~5-10 minutes (first time only)

### Step 4: Start the Container

```bash
docker-compose up -d
```

Or using docker directly:

```bash
docker run -d \
  --name nginx-active-health-check \
  -p 80:80 \
  -p 8080:8080 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  --restart unless-stopped \
  nginx-health-check:latest
```

### Step 5: Verify Load Balancer is Running

```bash
# Check container status
docker ps

# Check NGINX status
curl http://localhost:8080/nginx_status

# Check health check status (HTML dashboard)
curl http://localhost/upstream_check

# Or open in browser
open http://localhost/upstream_check
```

---

## 🔧 Part 2: Setup Backend Servers

Each backend server needs:
1. A web server (NGINX, Apache, Node.js, etc.)
2. A `/health` endpoint that returns HTTP 200

### NGINX Backend

**1. Install NGINX** (if not already installed):

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx -y

# RHEL/CentOS
sudo yum install nginx -y
```

**2. Configure NGINX**

**Quick Method**: Use the provided configuration file:

```bash
# Copy the provided backend-nginx.conf
sudo cp backend-nginx.conf /etc/nginx/nginx.conf
```

**Manual Method**: Create or edit `/etc/nginx/sites-available/default` (or `/etc/nginx/nginx.conf`):

```nginx
server {
    listen 80;
    server_name _;

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Your application
    location / {
        # Example: Simple response
        return 200 "Backend Server: $hostname\n";
        add_header Content-Type text/plain;
        
        # Or proxy to your actual application
        # proxy_pass http://localhost:3000;
    }
}
```

**3. Test and Restart**

```bash
# Test configuration
sudo nginx -t

# Restart NGINX
sudo systemctl restart nginx

# Test health endpoint
curl http://localhost/health
# Should return: healthy
```

**4. Verify from Load Balancer**

Quick Verification Commands

```bash
# 1. Check container status
docker ps

# 2. Check NGINX status endpoint
curl http://<nginx-lb-ip>:8080/nginx_status

# 3. Check health check status page
curl http://<nginx-lb-ip>/upstream_check

# 4. Test main endpoint (will fail until backends are configured)
curl http://<nginx-lb-ip>

# 5. View container logs
docker logs nginx-active-health-check

# 6. Check NGINX configuration syntax
docker exec nginx-active-health-check nginx -t
```

From your load balancer machine:

```bash
# Test connectivity to Backend 1
curl http://BACKEND1_IP/health

# Test connectivity to Backend 2  
curl http://BACKEND2_IP/health
```
---

## 🧪 Part 3: Testing

### Test 1: Basic Load Balancing

```bash
# Make multiple requests
for i in {1..10}; do
  curl http://LOAD_BALANCER_IP/
  sleep 1
done
```

You should see responses from different backend servers.

### Test 2: Health Check Status

Open in browser: `http://LOAD_BALANCER_IP/upstream_check`

You'll see a status page showing:
- **Server**: Backend IP:Port
- **Status**: up/down
- **Rise counts**: Successful checks
- **Fall counts**: Failed checks
- **Check type**: http
- **Check port**: 80

### Test 3: Backend Failure Simulation

**Stop one backend**:

```bash
# On Backend Server 1
sudo systemctl stop nginx  # or your web server
```

**Watch health check detect failure**:

```bash
# Monitor status page
watch -n 1 'curl -s http://LOAD_BALANCER_IP/upstream_check | grep -A5 "Backend"'

# Or continuously test
for i in {1..20}; do
  echo "Request $i:"
  curl http://LOAD_BALANCER_IP/
  sleep 2
done
```

After 3 failed checks (~9 seconds), traffic will only go to healthy backends.

**Restart backend**:

```bash
# On Backend Server 1
sudo systemctl start nginx
```

After 2 successful checks (~6 seconds), it will rejoin the pool.

### Test 4: Both Backends Down

```bash
# Stop both backends
# All requests will fail with 502 Bad Gateway
curl http://LOAD_BALANCER_IP/
```

---

## ⚙️ Configuration Reference

### Health Check Parameters

In `nginx.conf`, adjust the `check` directive:

```nginx
check interval=3000 rise=2 fall=3 timeout=2000 type=http;
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `interval` | 3000 | Milliseconds between checks (3s) |
| `rise` | 2 | Successful checks to mark server UP |
| `fall` | 3 | Failed checks to mark server DOWN |
| `timeout` | 2000 | Health check timeout in ms (2s) |
| `type` | http | Check type: tcp, http, ssl_hello, mysql, ajp |

**Examples**:

```nginx
# Very aggressive (1 second checks)
check interval=1000 rise=1 fall=2 timeout=1000 type=http;

# Lenient (10 second checks, allow more failures)
check interval=10000 rise=3 fall=5 timeout=5000 type=http;

# Fast failover (2 second checks, quick detection)
check interval=2000 rise=2 fall=2 timeout=1000 type=http;
```

### Adding More Backends

```nginx
upstream backend_servers {
    server 192.168.1.10:80 weight=1;
    server 192.168.1.11:80 weight=1;
    server 192.168.1.12:80 weight=1;  # Add more
    server 192.168.1.13:80 weight=2;  # Higher weight = more traffic
    
    check interval=3000 rise=2 fall=3 timeout=2000 type=http;
    check_http_send "HEAD /health HTTP/1.1\r\nHost: health-check\r\nConnection: close\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

### Load Balancing Algorithms

Add after the servers, before the `check` directive:

```nginx
upstream backend_servers {
    # Round Robin (default)
    server 192.168.1.10:80;
    server 192.168.1.11:80;
    
    # OR Least Connections
    least_conn;
    
    # OR IP Hash (sticky sessions)
    ip_hash;
    
    check interval=3000 rise=2 fall=3 timeout=2000 type=http;
}
```

### Custom Health Check Endpoint

```nginx
upstream backend_servers {
    server 192.168.1.10:80;
    server 192.168.1.11:80;
    
    check interval=3000 rise=2 fall=3 timeout=2000 type=http;
    check_http_send "GET /custom-health HTTP/1.1\r\nHost: health\r\nConnection: close\r\n\r\n";
    check_http_expect_alive http_2xx;
}
```

---

## 🔍 Monitoring & Troubleshooting

### View Logs

```bash
# Container logs
docker logs nginx-active-health-check

# Follow logs in real-time
docker logs -f nginx-active-health-check

# Last 100 lines
docker logs --tail 100 nginx-active-health-check
```

### Test NGINX Configuration

```bash
# Test config syntax
docker exec nginx-active-health-check nginx -t

# Reload configuration (without downtime)
docker exec nginx-active-health-check nginx -s reload
```

### Common Issues

**1. Health checks always failing**

```bash
# Verify backend is reachable from container
docker exec nginx-active-health-check curl -v http://BACKEND_IP/health

# Check backend actually responds to /health
curl http://BACKEND_IP/health
```

**2. Container won't start**

```bash
# Check logs
docker logs nginx-active-health-check

# Test config
docker run --rm -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx-health-check nginx -t
```

**3. Network connectivity issues**

```bash
# Check if containers can reach backends
docker exec nginx-active-health-check ping BACKEND_IP

# Verify firewall rules allow traffic
sudo iptables -L -n | grep 80
```

**4. DNS issues (using hostnames instead of IPs)**

If using hostnames in `nginx.conf`, ensure DNS resolution works:

```bash
docker exec nginx-active-health-check nslookup backend1.example.com
```

---

## 📊 Health Check Status Page

Access: `http://LOAD_BALANCER_IP/upstream_check`

### Reading the Status Page

```
Nginx http upstream check status

Check upstream server number: 2, generation: 1

Index | Upstream | Name           | Status | Rise | Fall | Type | Port
0     | backend  | 192.168.1.10:80| up     | 125  | 0    | http | 80
1     | backend  | 192.168.1.11:80| down   | 0    | 3    | http | 80
```

- **Status**: `up` = healthy, `down` = unhealthy
- **Rise**: Count of successful consecutive checks
- **Fall**: Count of failed consecutive checks
- **Type**: Health check type (http, tcp, etc.)

### Restrict Status Page Access

Edit `nginx.conf`:

```nginx
location /upstream_check {
    check_status;
    access_log off;
    
    # Only allow from specific IPs
    allow 192.168.1.0/24;
    allow 10.0.0.0/8;
    deny all;
}
```

---

## 🔐 Production Considerations

### 1. Enable SSL/TLS

Add SSL termination to the load balancer:

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    location / {
        proxy_pass http://backend_servers;
        # ... rest of config
    }
}
```

### 2. Security Headers

```nginx
location / {
    proxy_pass http://backend_servers;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

### 3. Rate Limiting

```nginx
http {
    # Define rate limit zone
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
    
    server {
        location / {
            limit_req zone=mylimit burst=20 nodelay;
            proxy_pass http://backend_servers;
        }
    }
}
```

### 4. Logging Best Practices

```nginx
# Custom log format with timing
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream: $upstream_addr '
                    'upstream_time: $upstream_response_time '
                    'request_time: $request_time';

access_log /var/log/nginx/access.log detailed;
```

### 5. Backup Configuration

```bash
# Create backups before changes
docker cp nginx-active-health-check:/etc/nginx/nginx.conf nginx.conf.backup.$(date +%Y%m%d)
```

---

## 🔄 Updating Configuration

### Method 1: Edit and Reload (No Downtime)

```bash
# 1. Edit nginx.conf
vim nginx.conf

# 2. Test configuration
docker run --rm -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx-health-check nginx -t

# 3. Restart container
docker-compose restart
```

### Method 2: Using Volume Mounts

In `docker-compose.yml`:

```yaml
services:
  nginx-lb:
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./logs:/var/log/nginx
```

Then reload without recreating:

```bash
# Edit nginx.conf
# Reload NGINX
docker exec nginx-active-health-check nginx -s reload
```

---

## 📦 Sharing with Teams

### Option 1: Push to Docker Registry

```bash
# Tag image
docker tag nginx-health-check:latest your-registry.com/nginx-health-check:latest

# Push
docker push your-registry.com/nginx-health-check:latest

# Team members pull and run
docker pull your-registry.com/nginx-health-check:latest
docker run -d -p 80:80 your-registry.com/nginx-health-check:latest
```

### Option 2: Save as TAR file

```bash
# Save image
docker save nginx-health-check:latest | gzip > nginx-health-check.tar.gz

# Transfer to other machines
scp nginx-health-check.tar.gz user@remote:/tmp/

# Load on other machine
gunzip -c nginx-health-check.tar.gz | docker load
```

### Option 3: Share Dockerfile

Share these files:
- `Dockerfile`
- `nginx.conf`
- `docker-compose.yml`
- This guide

Team members can build their own image:

```bash
docker build -t nginx-health-check:latest .
```

---

## 🎓 Quick Reference

### Essential Commands

```bash
# Build image
docker build -t nginx-health-check:latest .

# Start container
docker-compose up -d

# Stop container
docker-compose down

# View logs
docker logs -f nginx-active-health-check

# Reload config
docker exec nginx-active-health-check nginx -s reload

# Test config
docker exec nginx-active-health-check nginx -t

# Access shell
docker exec -it nginx-active-health-check bash
```

### URLs to Bookmark

- Health Status: `http://LOAD_BALANCER_IP/upstream_check`
- NGINX Status: `http://LOAD_BALANCER_IP:8080/nginx_status`
- Application: `http://LOAD_BALANCER_IP/`

---

## 🆘 Support & Resources

- **nginx_upstream_check_module**: https://github.com/yaoweibin/nginx_upstream_check_module
- **NGINX Docs**: https://nginx.org/en/docs/
- **Docker Docs**: https://docs.docker.com/

---

**You're all set! Your NGINX active health check load balancer is ready to use.** 🚀
