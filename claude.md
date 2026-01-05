# FastAPI + Cloudflare Tunnel Setup Guide

A complete guide to exposing a FastAPI application to the internet via Cloudflare Tunnel.

## Prerequisites

- Ubuntu VM
- Docker installed
- cloudflared installed
- Domain managed by Cloudflare (or nameservers pointed to Cloudflare)

---

## Step 1: Create the FastAPI Application

### 1.1 Create directory structure

```bash
mkdir -p ~/webapp/templates
cd ~/webapp
```

### 1.2 Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
fastapi
uvicorn[standard]
jinja2
EOF
```

### 1.3 Create main.py

```bash
cat > main.py << 'EOF'
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates

app = FastAPI()
templates = Jinja2Templates(directory="templates")

@app.get("/", response_class=HTMLResponse)
async def home(request: Request):
    return templates.TemplateResponse("index.html", {"request": request})
EOF
```

### 1.4 Create templates/index.html

```bash
cat > templates/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Hello World</title>
</head>
<body>
    <h1>Hello World!</h1>
    <p>FastAPI is running.</p>
</body>
</html>
EOF
```

### 1.5 Create Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

---

## Step 2: Build and Run with Docker

### 2.1 Build the image

```bash
cd ~/webapp
docker build -t webapp .
```

### 2.2 Run the container

```bash
docker run -d -p 8000:8000 --name webapp webapp
```

### 2.3 Test locally

```bash
curl localhost:8000
```

### Rebuilding after changes

```bash
docker stop webapp && docker rm webapp
docker build -t webapp . && docker run -d -p 8000:8000 --name webapp webapp
```

---

## Step 3: Domain Setup (One-time)

### If using GoDaddy (or other registrar) with Cloudflare:

1. Add domain to Cloudflare at https://dash.cloudflare.com (free plan)
2. Cloudflare provides nameservers (e.g., `bethany.ns.cloudflare.com`, `tosana.ns.cloudflare.com`)
3. Go to GoDaddy → Domain → Nameservers → Change to Cloudflare's nameservers
4. Wait for propagation (check with `dig NS yourdomain.com +short`)
5. Cloudflare dashboard will show domain as "Active" when ready

---

## Step 4: Cloudflare Tunnel Setup

### 4.1 Login to Cloudflare

```bash
cloudflared tunnel login
```

Opens a browser URL - authenticate and select your domain.

### 4.2 Create the tunnel

```bash
cloudflared tunnel create webapp
```

**Save the tunnel ID** (UUID) that's displayed.

### 4.3 Create config file

```bash
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL_ID>
credentials-file: /home/sean/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: yourdomain.com
    service: http://localhost:8000
  - service: http_status:404
EOF
```

Replace `<TUNNEL_ID>` with your actual tunnel ID (in both places).
Replace `yourdomain.com` with your actual domain.

### 4.4 Route DNS to tunnel

```bash
cloudflared tunnel route dns webapp yourdomain.com
```

If a DNS record already exists, delete it in Cloudflare dashboard first (DNS → Records).

### 4.5 Run the tunnel

```bash
cloudflared tunnel run webapp
```

---

## Step 5: Run Tunnel as a Service (Persistent)

To keep the tunnel running after closing terminal and auto-start on boot:

### 5.1 Copy config files to system location

The service runs as root, so config must be in `/etc/cloudflared/`:

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/
sudo cp ~/.cloudflared/*.json /etc/cloudflared/
```

### 5.2 Update credentials path in config

```bash
sudo sed -i 's|/home/sean/.cloudflared/|/etc/cloudflared/|g' /etc/cloudflared/config.yml
```

### 5.3 Install and start the service

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

### Useful commands

```bash
sudo systemctl status cloudflared    # Check status
sudo systemctl restart cloudflared   # Restart
sudo systemctl stop cloudflared      # Stop
journalctl -u cloudflared -f         # View logs
```

---

## Useful Commands Reference

### Tunnel management

```bash
cloudflared tunnel list              # List all tunnels
cloudflared tunnel info webapp       # Info about specific tunnel
cloudflared tunnel delete webapp     # Delete a tunnel
```

### Docker management

```bash
docker ps                            # Running containers
docker logs webapp                   # View app logs
docker exec -it webapp /bin/bash     # Shell into container
```

### Verify DNS propagation

```bash
dig NS yourdomain.com +short
dig yourdomain.com +short
```

---

## Security Notes

- Cloudflare Tunnel is outbound-only (no open ports on router)
- Your home IP is hidden behind Cloudflare
- Free DDoS protection and SSL/TLS included
- Consider adding Cloudflare Access for authentication (Zero Trust → Access → Applications)
- Keep Docker and cloudflared updated
- Consider isolating the server on a separate VLAN

---

## File Locations

| File               | Path                              |
| ------------------ | --------------------------------- |
| App code           | `~/webapp/`                       |
| Cloudflare cert    | `~/.cloudflared/cert.pem`         |
| Tunnel credentials | `~/.cloudflared/<TUNNEL_ID>.json` |
| Tunnel config      | `~/.cloudflared/config.yml`       |

---

## Tech Debt

- [ ] **Migrate secrets to Ansible Vault** - Currently using `vars/secrets.yml` (gitignored) for sensitive data like `tailscale_authkey`. Should encrypt with Ansible Vault for better security and ability to commit encrypted secrets.
