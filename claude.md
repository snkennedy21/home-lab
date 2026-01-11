# Kubernetes Playground - Infrastructure Automation

This repository automates the setup of Ubuntu VMs with Docker, Tailscale, and Cloudflare Tunnel for hosting web applications.

---

## Quick Start

### Prerequisites

1. Install Ansible on your local machine
2. Install the Docker collection:
   ```bash
   ansible-galaxy collection install community.docker
   ```
3. Configure your secrets in `vars/secrets.yml` (see Secrets section)

### Run the Playbooks

**Option 1: Full stack (everything at once)**
```bash
ansible-playbook playbooks/core-utilities.yml
```

**Option 2: Split workflow (for demos or staged setup)**
```bash
# Step 1: Base infrastructure + Tailscale (run before taking VM snapshot)
ansible-playbook playbooks/base-infrastructure.yml

# Step 2: Deploy webapp + Cloudflare Tunnel (run after snapshot revert)
ansible-playbook playbooks/deploy-webapp.yml
```

This provisions all hosts in `inventory/hosts.yml`.

---

## Repository Structure

```
.
├── ansible.cfg              # Ansible configuration
├── inventory/
│   └── hosts.yml            # Target servers
├── playbooks/
│   ├── core-utilities.yml   # Full stack (everything)
│   ├── base-infrastructure.yml  # Core utils + Tailscale only
│   └── deploy-webapp.yml    # Docker + App + Cloudflare Tunnel
├── vars/
│   └── secrets.yml          # Secrets (gitignored)
└── .gitignore
```

---

## What the Playbooks Install

### `base-infrastructure.yml` (run before snapshot)

| Component | Description |
|-----------|-------------|
| Core utilities | git, curl, wget, htop, vim, jq, tmux, etc. |
| Tailscale | Mesh VPN for remote SSH access |

### `deploy-webapp.yml` (run after snapshot revert)

| Component | Description |
|-----------|-------------|
| Docker | Docker CE with compose plugin |
| Node.js 20.x | Via NodeSource repository |
| Claude Code | `@anthropic-ai/claude-code` CLI |
| cloudflared | Cloudflare Tunnel daemon |
| FastAPI webapp | Hello World app in Docker container |

### `core-utilities.yml` (full stack)

Installs everything from both playbooks above in one run.

---

## Secrets Configuration

Create `vars/secrets.yml` with your credentials:

```yaml
---
# Local secrets - DO NOT COMMIT
# TODO: Migrate to Ansible Vault

tailscale_authkey: "tskey-auth-xxxxx"
cloudflare_tunnel_token: "eyJhIjoixxxxx"
```

This file is gitignored.

### Getting the Secrets

**Tailscale Auth Key:**
1. Open Tailscale menu bar icon → Admin console
2. Go to Settings → Keys
3. Generate auth key (enable "Reusable" for multiple machines)

**Cloudflare Tunnel Token:**
1. Go to https://one.dash.cloudflare.com
2. Networks → Tunnels → Create tunnel (or select existing)
3. Choose "Cloudflared" connector
4. Copy the token from the install command (`eyJ...`)
5. Configure public hostname: your-domain.com → http://localhost:8000

---

## Inventory

Edit `inventory/hosts.yml` to add your servers:

```yaml
---
all:
  hosts:
    myserver:
      ansible_host: 192.168.0.11
    VM-2:
      ansible_host: 192.168.0.6
```

---

## Accessing Your VMs

### Via Tailscale (from anywhere)

After Tailscale is installed, you can SSH from anywhere:

```bash
# Find Tailscale IP
tailscale ip -4

# SSH via Tailscale IP
ssh sean@100.x.x.x
```

View all machines: https://login.tailscale.com/admin/machines

### Via Cloudflare Tunnel

Your FastAPI app is exposed at your configured domain (e.g., `https://yourdomain.com`).

Check tunnel status in Cloudflare Zero Trust dashboard → Networks → Tunnels.

---

## Manual FastAPI Setup (Alternative)

If you prefer to set up manually instead of using Ansible:

### Create the Application

```bash
mkdir -p ~/webapp && cd ~/webapp

cat > main.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
EOF

cat > requirements.txt << 'EOF'
fastapi
uvicorn[standard]
EOF

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

### Build and Run

```bash
docker build -t webapp .
docker run -d -p 8000:8000 --name webapp webapp
curl localhost:8000
```

### Rebuild After Changes

```bash
docker stop webapp && docker rm webapp
docker build -t webapp . && docker run -d -p 8000:8000 --name webapp webapp
```

---

## Cloudflare Tunnel Setup (Dashboard Method)

The playbook uses the dashboard-managed tunnel approach (recommended):

1. **Create tunnel in dashboard:**
   - Go to Zero Trust → Networks → Tunnels
   - Create tunnel → Cloudflared → Name it → Save

2. **Get the token:**
   - Copy token from install command (`eyJ...`)
   - Add to `vars/secrets.yml`

3. **Configure public hostname:**
   - In tunnel config, add hostname
   - Domain: your-domain.com
   - Service: http://localhost:8000

4. **Run playbook:**
   ```bash
   ansible-playbook playbooks/core-utilities.yml
   ```

### Migrating Existing CLI Tunnels

If you have an existing tunnel created via `cloudflared tunnel create`:

1. Go to Zero Trust → Networks → Tunnels
2. Click on your tunnel → "Migrate"
3. Follow the wizard to migrate ingress rules
4. After migration, get the token from the dashboard

---

## Useful Commands

### Ansible

```bash
# Full stack
ansible-playbook playbooks/core-utilities.yml

# Split workflow
ansible-playbook playbooks/base-infrastructure.yml   # Tailscale + core utils
ansible-playbook playbooks/deploy-webapp.yml         # Docker + app + tunnel

# Target single host
ansible-playbook playbooks/deploy-webapp.yml --limit VM-2

# Test connectivity
ansible all -m ping
```

### Docker

```bash
docker ps                            # Running containers
docker logs webapp                   # View app logs
docker exec -it webapp /bin/bash     # Shell into container
```

### Tailscale

```bash
tailscale status                     # Show connected devices
tailscale ip -4                      # Show Tailscale IP
tailscale ping <hostname>            # Test connectivity
```

### Cloudflare Tunnel

```bash
sudo systemctl status cloudflared    # Check service status
sudo systemctl restart cloudflared   # Restart tunnel
journalctl -u cloudflared -f         # View logs
```

---

## Troubleshooting

### Clock Sync Issues

If you see "Release file is not valid yet" errors:

```bash
ssh sean@<ip> "sudo timedatectl set-ntp on && sudo systemctl restart systemd-timesyncd"
```

### Host Unreachable

1. Check VM is running
2. Verify SSH works: `ssh sean@<ip>`
3. Confirm IP in inventory is correct

### Tunnel Not Connecting

1. Check service status: `sudo systemctl status cloudflared`
2. View logs: `journalctl -u cloudflared -f`
3. Verify token is correct in `vars/secrets.yml`
4. Check Cloudflare dashboard for tunnel health

---

## Security Notes

- Cloudflare Tunnel is outbound-only (no open ports on router)
- Your home IP is hidden behind Cloudflare
- Free DDoS protection and SSL/TLS included
- Tailscale uses WireGuard encryption
- Consider adding Cloudflare Access for authentication
- Keep Docker, cloudflared, and Tailscale updated

---

## Tech Debt

- [ ] **Migrate secrets to Ansible Vault** - Currently using `vars/secrets.yml` (gitignored) for sensitive data. Should encrypt with Ansible Vault for better security and ability to commit encrypted secrets.
- [x] **Split playbook into roles** - Playbooks split into `base-infrastructure.yml` and `deploy-webapp.yml` for staged/demo workflows.
- [ ] **Add health checks** - Add verification tasks to confirm services are running correctly.
