# Part 2: Docker & Nginx Configuration Guide

## Step-by-Step Instructions for Configuring Services

**Prerequisites from Part 1:**
- VM is running
- You're SSH'd into the VM
- DNS records are configured and propagated

---

## Step 1: Update System and Install Docker

```bash
# Update package lists
sudo apt-get update

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (so you don't need sudo)
sudo usermod -aG docker $USER

# Apply group changes (or log out and back in)
newgrp docker

# Verify Docker is working
docker --version
```

**Expected output:** `Docker version 24.x.x, build...`

---

## Step 2: Create Project Directory Structure

```bash
# Create main directory
sudo mkdir -p /opt/ai-stack
cd /opt/ai-stack

# Create subdirectories for persistent data
sudo mkdir -p qdrant_data flowise_data langflow_data models

# Set permissions
sudo chown -R $USER:$USER /opt/ai-stack
```

---

## Step 3: Create docker-compose.yml

```bash
cat <<'EOF' > /opt/ai-stack/docker-compose.yml
version: "3"

services:
  qdrant:
    image: qdrant/qdrant
    restart: always
    container_name: qdrant
    ports:
      - "127.0.0.1:6333:6333"
    volumes:
      - ./qdrant_data:/qdrant/storage
    networks:
      - ai-network

  local-ai:
    image: localai/localai:latest
    restart: always
    container_name: local-ai
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - MODELS_PATH=/models
      - THREADS=4
      - PRELOAD_MODELS=[{"name": "multilingual-e5-large", "url": "github:go-skynet/model-gallery/huggingface.yaml"}]
    volumes:
      - ./models:/models
    networks:
      - ai-network

  flowise:
    image: flowiseai/flowise:latest
    restart: always
    container_name: flowise
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - PORT=3000
      - QDRANT_SERVER_URL=http://qdrant:6333
      - LOCALAI_BASE_URL=http://local-ai:8080/v1
    volumes:
      - ./flowise_data:/root/.flowise
    depends_on:
      - qdrant
      - local-ai
    networks:
      - ai-network

  langflow:
    image: langflowai/langflow:latest
    restart: always
    container_name: langflow
    ports:
      - "127.0.0.1:7860:7860"
    environment:
      - LANGFLOW_HOST=0.0.0.0
      - LANGFLOW_PORT=7860
      - LANGFLOW_DATABASE_URL=sqlite:////app/langflow/langflow.db
    volumes:
      - ./langflow_data:/app/langflow
    depends_on:
      - qdrant
      - local-ai
    networks:
      - ai-network

networks:
  ai-network:
    driver: bridge
EOF
```

**Key points:**
- All ports are bound to `127.0.0.1` (localhost only) - services are NOT directly accessible from outside
- All services use the same Docker network `ai-network` - they can talk to each other using container names
- Nginx will proxy external requests to these internal services

---

## Step 4: Start Docker Containers

```bash
cd /opt/ai-stack
docker compose up -d
```

**What this does:** Starts all 4 services in detached mode.

**Monitor the startup (especially LocalAI downloading the model):**
```bash
# Watch all logs
docker compose logs -f

# Watch only LocalAI (to see model download)
docker logs -f local-ai

# Press Ctrl+C to exit logs
```

**First startup will take 5-10 minutes** while LocalAI downloads the 2GB multilingual-e5-large model.

**Verify all containers are running:**
```bash
docker ps
```

You should see 4 containers: qdrant, local-ai, flowise, langflow.

---

## Step 5: Install Nginx and Certbot

```bash
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

---

## Step 6: Create Nginx Configuration for Flowise

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/flowise
server {
    server_name flowise.ptk.ge;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'
```

---

## Step 7: Create Nginx Configuration for Qdrant

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/qdrant
server {
    server_name qdrant.ptk.ge;

    location / {
        proxy_pass http://127.0.0.1:6333;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'
```

---

## Step 8: Create Nginx Configuration for Langflow

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/langflow
server {
    server_name langflow.ptk.ge;

    location / {
        proxy_pass http://127.0.0.1:7860;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'
```

---

## Step 9: Create Nginx Configuration for LocalAI

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/local-ai
server {
    server_name local-ai.ptk.ge;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        
        # Increase timeouts for AI processing
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }
}
EOF'
```

---

## Step 10: Enable All Nginx Sites

```bash
# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Enable all our sites
sudo ln -s /etc/nginx/sites-available/flowise /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/qdrant /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/langflow /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/local-ai /etc/nginx/sites-enabled/

# Test Nginx configuration
sudo nginx -t
```

**Expected output:** `syntax is ok` and `test is successful`

```bash
# Restart Nginx
sudo systemctl restart nginx
```

---

## Step 11: Register with Let's Encrypt

```bash
# Register your email (only needed once)
sudo certbot register --email paata.koridze@gmail.com --agree-tos --no-eff-email
```

---

## Step 12: Get SSL Certificates for All Subdomains

Run these commands **one at a time**:

```bash
# Certificate for Flowise
sudo certbot --nginx -d flowise.ptk.ge --non-interactive --agree-tos -m paata.koridze@gmail.com

# Certificate for Qdrant
sudo certbot --nginx -d qdrant.ptk.ge --non-interactive --agree-tos -m paata.koridze@gmail.com

# Certificate for Langflow
sudo certbot --nginx -d langflow.ptk.ge --non-interactive --agree-tos -m paata.koridze@gmail.com

# Certificate for LocalAI
sudo certbot --nginx -d local-ai.ptk.ge --non-interactive --agree-tos -m paata.koridze@gmail.com
```

Each command should output: `Successfully deployed certificate`

**If you get DNS errors:**
- Wait 10 more minutes for DNS propagation
- Verify DNS with: `nslookup flowise.ptk.ge`

---

## Step 13: Verify Everything is Working

### Test from outside the VM:

```bash
# From your local machine, test each endpoint:
curl https://flowise.ptk.ge
curl https://qdrant.ptk.ge
curl https://langflow.ptk.ge
curl https://local-ai.ptk.ge/v1/models
```

### Test in browser:

Open these URLs:
- https://flowise.ptk.ge (Flowise UI)
- https://qdrant.ptk.ge/dashboard (Qdrant dashboard)
- https://langflow.ptk.ge (Langflow UI)
- https://local-ai.ptk.ge/v1/models (LocalAI models list)

All should load with valid SSL certificates (green padlock).

---

## Step 14: Configure Auto-Renewal for SSL Certificates

```bash
# Test auto-renewal
sudo certbot renew --dry-run
```

**Expected output:** `Congratulations, all simulated renewals succeeded`

Certbot automatically sets up a cron job to renew certificates. Verify:
```bash
sudo systemctl status certbot.timer
```

---

## How Services Communicate

### Internal Communication (Docker network):
- Flowise → Qdrant: `http://qdrant:6333`
- Flowise → LocalAI: `http://local-ai:8080/v1`
- Langflow → Qdrant: `http://qdrant:6333`
- Langflow → LocalAI: `http://local-ai:8080/v1`

### External Access (via Nginx):
- Flowise: `https://flowise.ptk.ge` → proxied to `http://127.0.0.1:3000`
- Qdrant: `https://qdrant.ptk.ge` → proxied to `http://127.0.0.1:6333`
- Langflow: `https://langflow.ptk.ge` → proxied to `http://127.0.0.1:7860`
- LocalAI: `https://local-ai.ptk.ge` → proxied to `http://127.0.0.1:8080`

**Security:** Direct access to ports 3000, 6333, 7860, 8080 from outside is blocked because they're bound to 127.0.0.1.

---

## Configuration for Flowise

When setting up embeddings in Flowise:

1. Use the **OpenAI Embeddings** node
2. **Base Path:** `http://local-ai:8080/v1`
3. **Model Name:** `multilingual-e5-large`
4. **API Key:** `local` (any value works)

When connecting to Qdrant:

1. Use the **Qdrant** vector store node
2. **Qdrant URL:** `http://qdrant:6333`
3. **Collection Name:** Your choice (e.g., `my_docs`)

---

## Configuration for Langflow

When setting up embeddings in Langflow:

1. Add a **Custom API Embeddings** component
2. **API URL:** `http://local-ai:8080/v1/embeddings`
3. **Model:** `multilingual-e5-large`

When connecting to Qdrant:

1. Add a **Qdrant** vector store component
2. **URL:** `http://qdrant:6333`
3. **Collection Name:** Your choice (can use same as Flowise)

---

## Useful Commands

### View Docker logs:
```bash
docker compose logs -f               # All services
docker logs -f flowise               # Just Flowise
docker logs -f local-ai              # Just LocalAI
docker logs -f qdrant                # Just Qdrant
docker logs -f langflow              # Just Langflow
```

### Restart services:
```bash
docker compose restart               # Restart all
docker restart flowise               # Restart just Flowise
```

### Stop all services:
```bash
cd /opt/ai-stack
docker compose down
```

### Start all services:
```bash
cd /opt/ai-stack
docker compose up -d
```

### Check Nginx status:
```bash
sudo systemctl status nginx
sudo nginx -t                        # Test config
sudo systemctl restart nginx         # Restart Nginx
```

### View Nginx logs:
```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

---

## Troubleshooting

### If a subdomain shows 502 Bad Gateway:
1. Check if the Docker container is running: `docker ps`
2. Check container logs: `docker logs [container-name]`
3. Check if the port is listening: `sudo netstat -tlnp | grep [port]`
4. Restart the container: `docker restart [container-name]`

### If SSL certificate fails:
1. Verify DNS is working: `nslookup flowise.ptk.ge`
2. Check firewall allows port 80: `sudo ufw status` (should be inactive or allow 80/443)
3. Check Nginx is running: `sudo systemctl status nginx`
4. Try manual certificate: `sudo certbot certonly --nginx -d flowise.ptk.ge`

### If LocalAI is slow or not responding:
1. Check if model downloaded: `ls -lh /opt/ai-stack/models/`
2. Check LocalAI logs: `docker logs local-ai`
3. The first request after startup takes ~30 seconds while model loads into memory

### If services can't reach each other:
1. Verify Docker network: `docker network ls` (should show ai-network)
2. Check container network: `docker inspect flowise | grep NetworkMode`
3. Test connectivity: `docker exec flowise ping qdrant`

---

## Security Recommendations

1. **Restrict Qdrant access** (optional):
   - Qdrant dashboard at https://qdrant.ptk.ge is publicly accessible
   - Consider adding basic auth or IP whitelist to the Nginx config

2. **Add authentication to Flowise**:
   - Set environment variables in docker-compose.yml:
   ```yaml
   - FLOWISE_USERNAME=your_username
   - FLOWISE_PASSWORD=your_password
   ```

3. **Regular updates**:
   ```bash
   cd /opt/ai-stack
   docker compose pull        # Pull latest images
   docker compose up -d       # Recreate containers
   ```

---

## Backup Important Data

Create backups of:
```bash
# Backup all data directories
sudo tar -czf ai-stack-backup-$(date +%Y%m%d).tar.gz \
  /opt/ai-stack/qdrant_data \
  /opt/ai-stack/flowise_data \
  /opt/ai-stack/langflow_data \
  /opt/ai-stack/models
```

Store the backup file somewhere safe (Google Cloud Storage, etc.).

---

## Complete! 🎉

Your AI stack is now running with:

- ✅ Flowise at https://flowise.ptk.ge
- ✅ Qdrant at https://qdrant.ptk.ge
- ✅ Langflow at https://langflow.ptk.ge
- ✅ LocalAI at https://local-ai.ptk.ge

All services can communicate internally and are accessible externally via HTTPS with valid SSL certificates.
