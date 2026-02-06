#https://claude.ai/chat/f4cc9b92-8ad3-469f-800d-7f93e315736f
#ptkapigoogl

# AI Stack Deployment on Google Cloud Platform

A complete, production-ready AI infrastructure stack featuring Flowise, Langflow, Qdrant vector database, and LocalAI embeddings, all deployed on Google Cloud Platform with SSL-secured subdomains.

## 🌟 Overview

This repository contains infrastructure-as-code and configuration files for deploying a comprehensive AI development stack on GCP. The stack enables you to build, test, and deploy AI workflows using low-code tools (Flowise and Langflow), vector search (Qdrant), and local embeddings generation (LocalAI).

### What This Stack Provides

- **Flowise** (`https://flowise.ptk.ge`) - Low-code tool for building LLM applications with a drag-and-drop interface
- **Langflow** (`https://langflow.ptk.ge`) - Visual framework for building multi-agent and RAG applications
- **Qdrant** (`https://qdrant.ptk.ge`) - High-performance vector database for similarity search and AI applications
- **LocalAI** (`https://local-ai.ptk.ge`) - Self-hosted, privacy-focused alternative to OpenAI for embeddings generation

All services are:
- ✅ Accessible via HTTPS with valid SSL certificates
- ✅ Running in Docker containers for easy management
- ✅ Connected via a private network for inter-service communication
- ✅ Secured behind Nginx reverse proxy
- ✅ Auto-restarting on failures
- ✅ Persistently storing data across restarts

## 🏗️ Architecture

```
Internet
   │
   ├─→ https://flowise.ptk.ge ──────┐
   ├─→ https://langflow.ptk.ge ─────┤
   ├─→ https://qdrant.ptk.ge ───────┼─→ Nginx Reverse Proxy
   └─→ https://local-ai.ptk.ge ─────┘         │
                                               ↓
                                    Docker Network (ai-network)
                                               │
                    ┌──────────────────────────┼───────────────────────────┐
                    │                          │                           │
              ┌─────▼─────┐            ┌──────▼──────┐            ┌───────▼────────┐
              │  Flowise  │            │  Langflow   │            │    Qdrant      │
              │ :3000     │            │  :7860      │            │    :6333       │
              └─────┬─────┘            └──────┬──────┘            └───────▲────────┘
                    │                          │                           │
                    └──────────────────────────┴───────────────────────────┘
                                               │
                                        ┌──────▼──────┐
                                        │  LocalAI    │
                                        │  :8080      │
                                        └─────────────┘
```

### Network Architecture

**External Access (via HTTPS):**
- All services are accessible through Nginx reverse proxy with SSL certificates
- Direct port access is blocked; only Nginx can reach the services

**Internal Communication (Docker network):**
- Services communicate using Docker container names (e.g., `http://qdrant:6333`)
- All containers share the `ai-network` bridge network
- No external exposure of internal ports

## 📋 Prerequisites

- **Google Cloud Account** with billing enabled
- **Domain name** (in this setup: `ptk.ge`)
- **DNS access** to create A records
- **gcloud CLI** installed (or use Cloud Console)
- **Basic knowledge** of Docker, Nginx, and Linux

### Minimum VM Requirements

- **Machine Type:** e2-standard-4 (4 vCPUs, 16GB RAM)
- **Storage:** 100GB SSD
- **OS:** Ubuntu 22.04 LTS
- **Network:** Custom VPC with firewall rules

> **Note:** Smaller instances may work but will be slow, especially when LocalAI loads the 2GB embedding model.

## 🚀 Quick Start

### Step 1: Clone This Repository

```bash
git clone https://github.com/yourusername/ai-stack-gcp.git
cd ai-stack-gcp
```

### Step 2: Follow the Deployment Guides

This repository includes two comprehensive deployment guides:

1. **[PART1_GCP_VM_SETUP.md](./PART1_GCP_VM_SETUP.md)** - Create and configure your GCP VM
2. **[PART2_DOCKER_NGINX_CONFIG.md](./PART2_DOCKER_NGINX_CONFIG.md)** - Install Docker, configure Nginx, and deploy services

Each guide includes:
- Step-by-step terminal commands
- Explanations of what each command does
- Verification steps
- Troubleshooting tips

### Step 3: Configure DNS

Point these subdomains to your VM's external IP:

```
flowise.ptk.ge    → A record → [YOUR_VM_IP]
langflow.ptk.ge   → A record → [YOUR_VM_IP]
qdrant.ptk.ge     → A record → [YOUR_VM_IP]
local-ai.ptk.ge   → A record → [YOUR_VM_IP]
```

### Step 4: Deploy!

Once your VM is running and DNS is configured, SSH into the VM and follow Part 2 of the guide.

## 📁 Repository Structure

```
ai-stack-gcp/
├── README.md                       # This file
├── PART1_GCP_VM_SETUP.md          # GCP VM creation guide
├── PART2_DOCKER_NGINX_CONFIG.md   # Docker & Nginx setup guide
├── docker-compose.yml              # Docker Compose configuration
├── nginx/                          # Nginx configuration files
│   ├── flowise.conf               # Flowise reverse proxy config
│   ├── langflow.conf              # Langflow reverse proxy config
│   ├── qdrant.conf                # Qdrant reverse proxy config
│   └── local-ai.conf              # LocalAI reverse proxy config
├── scripts/                        # Utility scripts
│   ├── backup.sh                  # Backup script for data
│   ├── update.sh                  # Update all Docker images
│   └── health-check.sh            # Check all services status
└── docs/                           # Additional documentation
    ├── TROUBLESHOOTING.md         # Common issues and solutions
    ├── SECURITY.md                # Security best practices
    └── USAGE_EXAMPLES.md          # How to use each service
```

## 🔧 Configuration Files

### docker-compose.yml

The main configuration file that defines all services. Key features:

- **Qdrant** - Vector database with persistent storage
- **LocalAI** - Self-hosted embeddings with automatic model download
- **Flowise** - Configured to use LocalAI and Qdrant
- **Langflow** - Configured to use LocalAI and Qdrant

All services:
- Restart automatically on failure
- Bind to localhost (127.0.0.1) for security
- Share a common Docker network
- Persist data in local volumes

### Nginx Configurations

Each service has its own Nginx server block in `/etc/nginx/sites-available/`:

- Reverse proxy to Docker containers
- WebSocket support (for Flowise and Langflow)
- SSL certificate integration
- Proper headers for security

## 🔐 Security Features

### Network Security

- All Docker ports bound to `127.0.0.1` (not accessible externally)
- GCP firewall rules limit access to necessary ports only
- Custom VPC network isolates the VM

### SSL/TLS

- Automatic SSL certificates via Let's Encrypt
- Auto-renewal configured via Certbot
- HTTPS enforced for all services

### Access Control

- Qdrant dashboard publicly accessible (consider adding auth)
- Flowise and Langflow can be protected with environment variables
- SSH access only via GCP IAM-controlled keys

### Recommendations

1. **Enable Flowise authentication:**
   ```yaml
   environment:
     - FLOWISE_USERNAME=your_username
     - FLOWISE_PASSWORD=your_password
   ```

2. **Restrict Qdrant access** by adding basic auth to Nginx:
   ```nginx
   auth_basic "Restricted Access";
   auth_basic_user_file /etc/nginx/.htpasswd;
   ```

3. **Regular security updates:**
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   docker compose pull && docker compose up -d
   ```

## 📊 Service Details

### Flowise (Port 3000)

**Purpose:** Build LLM apps with a visual interface

**Access:** https://flowise.ptk.ge

**Features:**
- Drag-and-drop chatflow builder
- Pre-built templates for RAG, agents, and more
- Integration with Qdrant and LocalAI

**Using LocalAI in Flowise:**
1. Add an "OpenAI Embeddings" node
2. Set Base Path: `http://local-ai:8080/v1`
3. Set Model Name: `multilingual-e5-large`
4. API Key: `local` (any value works)

**Using Qdrant in Flowise:**
1. Add a "Qdrant" vector store node
2. Qdrant URL: `http://qdrant:6333`
3. Collection Name: Your choice

### Langflow (Port 7860)

**Purpose:** Build AI workflows and agents visually

**Access:** https://langflow.ptk.ge

**Features:**
- Component-based flow builder
- Python code editing inline
- Multi-agent orchestration
- Integration with Qdrant and LocalAI

**Using LocalAI in Langflow:**
1. Add a "Custom API" embeddings component
2. API URL: `http://local-ai:8080/v1/embeddings`
3. Model: `multilingual-e5-large`

**Using Qdrant in Langflow:**
1. Add a "Qdrant" vector store component
2. URL: `http://qdrant:6333`
3. Collection Name: Your choice

### Qdrant (Port 6333)

**Purpose:** High-performance vector database

**Access:** https://qdrant.ptk.ge

**Features:**
- Web dashboard for managing collections
- REST and gRPC APIs
- Efficient similarity search
- Persistent storage

**Collections:**
- Flowise and Langflow can share collections
- Data persisted in `/opt/ai-stack/qdrant_data`

### LocalAI (Port 8080)

**Purpose:** Self-hosted embeddings generation

**Access:** https://local-ai.ptk.ge

**Model:** `multilingual-e5-large` (2GB, auto-downloaded on first start)

**Features:**
- OpenAI-compatible API
- Privacy-focused (no data sent to external services)
- Support for 100+ languages

**API Endpoints:**
- `/v1/embeddings` - Generate embeddings
- `/v1/models` - List available models

**First Startup:**
The first time you start LocalAI, it downloads the model (~2GB). This takes 5-10 minutes. Monitor progress:
```bash
docker logs -f local-ai
```

## 💾 Data Persistence

All services store data in local directories:

```
/opt/ai-stack/
├── qdrant_data/      # Vector database collections
├── flowise_data/     # Flowise flows and credentials
├── langflow_data/    # Langflow flows and database
└── models/           # LocalAI embedding models
```

### Backup

Create a backup:
```bash
sudo tar -czf ai-stack-backup-$(date +%Y%m%d).tar.gz \
  /opt/ai-stack/qdrant_data \
  /opt/ai-stack/flowise_data \
  /opt/ai-stack/langflow_data \
  /opt/ai-stack/models
```

Restore from backup:
```bash
sudo tar -xzf ai-stack-backup-20250206.tar.gz -C /
```

## 🛠️ Management Commands

### Start all services
```bash
cd /opt/ai-stack
docker compose up -d
```

### Stop all services
```bash
docker compose down
```

### Restart a specific service
```bash
docker restart flowise
docker restart langflow
docker restart qdrant
docker restart local-ai
```

### View logs
```bash
docker compose logs -f              # All services
docker logs -f flowise              # Specific service
```

### Update all services
```bash
docker compose pull                 # Pull latest images
docker compose up -d                # Recreate containers
```

### Check service status
```bash
docker ps                           # Running containers
docker compose ps                   # Stack status
```

### Nginx management
```bash
sudo systemctl status nginx         # Check status
sudo systemctl restart nginx        # Restart Nginx
sudo nginx -t                       # Test configuration
sudo tail -f /var/log/nginx/error.log  # View error logs
```

## 🐛 Troubleshooting

### Service shows 502 Bad Gateway

1. Check if container is running: `docker ps`
2. Check container logs: `docker logs [container-name]`
3. Restart the container: `docker restart [container-name]`

### SSL certificate fails

1. Verify DNS: `nslookup flowise.ptk.ge`
2. Check Nginx is running: `sudo systemctl status nginx`
3. Check port 80 is open in GCP firewall
4. Retry: `sudo certbot --nginx -d flowise.ptk.ge`

### LocalAI is slow or times out

1. First request takes ~30 seconds (model loading)
2. Check if model downloaded: `ls -lh /opt/ai-stack/models/`
3. Increase VM resources if needed
4. Check logs: `docker logs local-ai`

### Services can't communicate

1. Verify Docker network: `docker network inspect ai-network`
2. Check all containers are on the network: `docker inspect flowise | grep NetworkMode`
3. Test connectivity: `docker exec flowise ping qdrant`

### Out of disk space

1. Check disk usage: `df -h`
2. Clean Docker images: `docker system prune -a`
3. Expand boot disk in GCP Console

## 💰 Cost Estimation

**Monthly costs for this setup:**

| Resource | Cost |
|----------|------|
| e2-standard-4 VM (us-central1) | ~$120 |
| 100GB SSD persistent disk | ~$17 |
| Network egress (estimate) | ~$5-20 |
| **Total** | **~$140-160/month** |

**Cost optimization tips:**

1. **Stop the VM when not in use:**
   ```bash
   gcloud compute instances stop vm1-ai-engine --zone=us-central1-a
   ```

2. **Use preemptible instances** (not recommended for production):
   Add `--preemptible` flag when creating the VM (saves ~60%)

3. **Use smaller machine type** for testing:
   `e2-standard-2` (2 vCPUs, 8GB RAM) costs ~$60/month but may be slow

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### Areas for contribution:
- Additional service integrations
- Security improvements
- Monitoring and alerting setup
- Performance optimizations
- Documentation improvements

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙏 Acknowledgments

- [Flowise](https://github.com/FlowiseAI/Flowise) - Low-code LLM apps builder
- [Langflow](https://github.com/logspace-ai/langflow) - Visual AI workflow builder
- [Qdrant](https://github.com/qdrant/qdrant) - Vector database
- [LocalAI](https://github.com/mudler/LocalAI) - Self-hosted AI
- [Let's Encrypt](https://letsencrypt.org/) - Free SSL certificates

## 📞 Support

If you encounter issues:

1. Check the [Troubleshooting Guide](./PART2_DOCKER_NGINX_CONFIG.md#troubleshooting)
2. Review service logs: `docker compose logs`
3. Open an issue in this repository
4. Contact: paata.koridze@gmail.com

## 🔗 Useful Links

- [Flowise Documentation](https://docs.flowiseai.com/)
- [Langflow Documentation](https://docs.langflow.org/)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [LocalAI Documentation](https://localai.io/)
- [Google Cloud Documentation](https://cloud.google.com/docs)

## 🗺️ Roadmap

- [ ] Add monitoring with Prometheus and Grafana
- [ ] Implement automated backups to Google Cloud Storage
- [ ] Add n8n for workflow automation
- [ ] Integrate Ollama for local LLM inference
- [ ] Add authentication layer for all services
- [ ] Create Terraform scripts for infrastructure-as-code
- [ ] Add CI/CD pipeline for updates
- [ ] Implement rate limiting and DDoS protection

---

**Built with ❤️ for the AI community**

Last updated: February 6, 2026
