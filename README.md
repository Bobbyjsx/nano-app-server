# Nano App Server

> **Note:** This project is a stripped-down, lightweight version of the main [ubuntu-server-iaac](https://github.com/Bobbyjsx/ubuntu-server-iaac) repository, optimized for single-project environments.

A lightweight, automated infrastructure-as-code (IaC) template designed to spin up a complete, secure application environment on a single small VM (like a GCP `e2-micro` or AWS `t3.nano`). 

Rather than a bulky "homelab", this is purpose-built to run an entire project front-to-back per instance, minimizing costs while maximizing security and convenience.

## 🚀 Features

- **Automated Provisioning:** One command (`make bootstrap`) hardens the server, installs Docker, and launches your services via Ansible.
- **Cloudflare Tunnels:** Fully hidden from the public internet for web interfaces. No open HTTP ports (80/443 are closed). Cloudflare handles SSL termination and reverse proxies directly to your internal containers.
- **PgBouncer & PostgreSQL:** A robust, production-ready database layer. PgBouncer runs in transaction mode, allowing secure connection pooling for serverless APIs (like Google Cloud Run or Vercel).
- **GoTrue Auth:** A powerful, internal-only authentication API powered by Supabase's GoTrue. Automatic schema initialization is handled via `init-db.sql`.
- **Portainer UI:** Manage your Docker containers visually through a web interface.
- **Security-First:** 
  - Database access is locked down by **UFW**. You can securely whitelist your external API's IP address directly from your `.env` file!
  - Tailscale VPN integration included for secure internal local development access (e.g. TablePlus or DBeaver).
  - Fail2ban configured automatically.

## 📦 Stack Architecture

```text
[ External API (Vercel/Cloud Run) ]    [ Cloudflare Edge (SSL Termination) ]
            │                                     │
            ▼ (Port 6432)                         ▼
     [ UFW Firewall Whitelist ]           [ Cloudflare Tunnels ]
            │                                     │
            ▼                                     ├── auth.domain.com -> GoTrue (HTTP 9999)
[ Internal Docker Network ]                       └── portainer.domain.com -> Portainer (HTTP 9000)
    ├── PgBouncer (Port 6432)
    │       │
    │       ▼
    ├── PostgreSQL (5432, Bound to localhost)
    ├── GoTrue
    └── Portainer
```

## 🛠️ Quick Start

### 1. Prerequisites
- A fresh VM (Ubuntu or Debian).
- A Cloudflare account with a domain.
- A Cloudflare Tunnel Token.
- Ansible installed on your local machine (`brew install ansible` or `apt install ansible`).

### 2. Configuration
Clone this repository and set up your environment variables:

```bash
# Copy the example environment file
make setup
```

Edit the generated `.env` file and fill in your target server details, application secrets, and Cloudflare token:

```ini
# Server Configuration
SERVER_IP=192.168.1.100
SERVER_USER=ubuntu
SSH_KEY_PATH=~/.ssh/id_ed25519
TAILSCALE_AUTH_KEY=tskey-auth-...

# Application Secrets
POSTGRES_USER=keysentry_admin
POSTGRES_PASSWORD=your_secure_postgres_password
POSTGRES_DB=postgres
API_EXTERNAL_URL=https://auth.yourdomain.com
GOTRUE_SITE_URL=https://app.yourdomain.com
GOTRUE_JWT_SECRET=your_super_secret_jwt_token_must_be_at_least_32_characters
CLOUDFLARE_TUNNEL_TOKEN=ey...

# Whitelist for PgBouncer (Leave blank if you don't want to expose it publicly)
API_WHITELIST_IP=10.218.0.2
```

### 3. Deployment
Run the bootstrap command. This will connect to your server, apply security rules, install Docker, and spin up your `docker-compose` environment.

```bash
make bootstrap
```

*Note: You may be prompted for your SSH password or sudo password during the Ansible provisioning step.*

### 4. Cloudflare Configuration
In your Cloudflare Zero Trust Dashboard:
1. Ensure your tunnel is showing as "Active".
2. Add Public Hostnames pointing to your internal Docker services:
   - `auth.yourdomain.com` -> `http://gotrue:9999`
   - `portainer.yourdomain.com` -> `http://portainer:9000`

### 5. Verification
Once DNS propagates, you can navigate to:
- `https://portainer.yourdomain.com` (Set up your initial admin password)
- `https://auth.yourdomain.com/health` (Verify GoTrue is running)

## 🔧 Adding Your Own App

To add your own frontend or backend services, simply:
1. Open `docker/compose.yml` and add your service.
2. Re-run `make bootstrap` or log into Portainer to redeploy.
3. Add a new Public Hostname in your Cloudflare Zero Trust dashboard pointing to your new container's internal HTTP port.
