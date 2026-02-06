# Part 1: GCP VM Setup Guide

## Step-by-Step Instructions for Creating VM in Google Cloud Platform

### Prerequisites
- Google Cloud account with billing enabled
- `gcloud` CLI installed (or use Google Cloud Console)
- Project ID: `ai24-flowise`

---

## Step 1: Create Custom VPC Network

```bash
gcloud compute networks create ai-network \
    --project=ai24-flowise \
    --subnet-mode=custom \
    --bgp-routing-mode=regional
```

**What this does:** Creates a custom Virtual Private Cloud network that gives you full control over IP addressing and subnets.

---

## Step 2: Create Subnet in us-central1 Region

```bash
gcloud compute networks subnets create ai-subnet \
    --project=ai24-flowise \
    --network=ai-network \
    --region=us-central1 \
    --range=10.0.1.0/24
```

**What this does:** Creates a subnet with IP range 10.0.1.0/24 (256 IP addresses) in the us-central1 region.

---

## Step 3: Create Firewall Rules (Open Required Ports)

```bash
gcloud compute firewall-rules create allow-ai-access \
    --project=ai24-flowise \
    --network=ai-network \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:22,tcp:80,tcp:443,tcp:3000,tcp:6333,tcp:7860,tcp:8080 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=ai-engine
```

**What this does:** Opens the following ports:
- **22:** SSH access
- **80:** HTTP (for Nginx before SSL)
- **443:** HTTPS (for Nginx with SSL)
- **3000:** Flowise (internal, will be proxied)
- **6333:** Qdrant (internal, will be proxied)
- **7860:** Langflow (internal, will be proxied)
- **8080:** LocalAI (internal, will be proxied)

---

## Step 4: Create the VM Instance

```bash
gcloud compute instances create vm1-ai-engine \
    --project=ai24-flowise \
    --zone=us-central1-a \
    --machine-type=e2-standard-4 \
    --network=ai-network \
    --subnet=ai-subnet \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=100GB \
    --boot-disk-type=pd-balanced \
    --tags=ai-engine
```

**Important changes from your original script:**
- **Machine type:** Changed to `e2-standard-4` (4 vCPUs, 16GB RAM) - required for running 4 AI services
- **Boot disk:** Increased to 100GB to accommodate models and data
- **No startup script yet:** We'll configure everything manually in Part 2 for better control

**What this does:** Creates a VM with:
- Ubuntu 22.04 LTS
- 4 vCPUs and 16GB RAM
- 100GB storage
- Connected to your custom network

---

## Step 5: Get the External IP Address

After the VM is created, get its external IP:

```bash
gcloud compute instances describe vm1-ai-engine \
    --project=ai24-flowise \
    --zone=us-central1-a \
    --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

**Save this IP address** - you'll need it for DNS configuration.

---

## Step 6: Configure DNS Records

Go to your DNS provider (where you manage ptk.ge) and create these A records:

| Subdomain | Type | Value (your VM IP) | TTL |
|-----------|------|-------------------|-----|
| flowise.ptk.ge | A | [YOUR_VM_IP] | 300 |
| qdrant.ptk.ge | A | [YOUR_VM_IP] | 300 |
| langflow.ptk.ge | A | [YOUR_VM_IP] | 300 |
| local-ai.ptk.ge | A | [YOUR_VM_IP] | 300 |

**Important:** Wait 5-10 minutes for DNS propagation before proceeding to Part 2.

You can verify DNS is working:
```bash
nslookup flowise.ptk.ge
nslookup qdrant.ptk.ge
nslookup langflow.ptk.ge
nslookup local-ai.ptk.ge
```

Each should return your VM's IP address.

---

## Step 7: SSH into Your VM

```bash
gcloud compute ssh vm1-ai-engine \
    --project=ai24-flowise \
    --zone=us-central1-a
```

Or use the SSH button in Google Cloud Console.

---

## Verification Checklist

Before proceeding to Part 2, verify:

- [ ] VM is running (check in Cloud Console)
- [ ] You can SSH into the VM
- [ ] All 4 DNS records point to the VM IP
- [ ] DNS records have propagated (use nslookup)
- [ ] Firewall rules are active (check Cloud Console > VPC Network > Firewall)

---

## Next Steps

Once your VM is created and DNS is configured, proceed to **Part 2: Docker & Nginx Configuration** where you'll:
- Install Docker
- Create docker-compose.yml
- Install and configure Nginx
- Set up SSL certificates for all subdomains
- Start all services

---

## Cost Estimation

**Estimated monthly cost for this setup:**
- e2-standard-4 VM: ~$120/month
- 100GB balanced persistent disk: ~$17/month
- Network egress: Variable (typically $5-20/month)

**Total: ~$140-160/month**

To reduce costs, you can shut down the VM when not in use:
```bash
gcloud compute instances stop vm1-ai-engine --zone=us-central1-a
```

Start it again:
```bash
gcloud compute instances start vm1-ai-engine --zone=us-central1-a
```
