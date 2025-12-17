# Businka Cloud Infrastructure

Nginx configuration and deployment setup for businka.cloud and its subdomains.

## Repository Structure

```
├── .github/workflows/
│   └── deploy-nginx.yml          # GitHub Actions deployment workflow
└── sites/
    ├── businka.cloud.conf        # Main domain Nginx config
    └── game.businka.cloud.conf   # Game subdomain Nginx config
```

## Adding a New Subdomain

### 1. DNS Configuration (Google Cloud DNS)

Go to [Google Cloud DNS Console](https://console.cloud.google.com/net-services/dns/zones/businka-cloud/details?project=businka-cloud) and add a CNAME record:

| Field | Value |
|-------|-------|
| DNS name | `<subdomain>.businka.cloud` |
| Resource record type | `CNAME` |
| TTL | `300` |
| Canonical name | `businka.cloud.` |

**Current DNS records:**

| Name | Type | Value |
|------|------|-------|
| `businka.cloud` | A | Server IP |
| `game.businka.cloud` | CNAME | `businka.cloud.` |

### 2. Server Setup for HTTPS

After adding DNS records and deploying the Nginx config, SSH into the server to obtain SSL certificates.

#### First-time certificate setup (when certificate doesn't exist yet)

Since Nginx config references certificates that don't exist yet, use one of these methods:

**Option A: Standalone mode (requires brief downtime)**

```bash
# Stop Nginx to free up port 80
sudo systemctl stop nginx

# Obtain certificate
sudo certbot certonly --standalone -d <subdomain>.businka.cloud

# Start Nginx
sudo systemctl start nginx
```

**Option B: Keep other sites running (no downtime)**

```bash
# Temporarily disable the new subdomain config
sudo rm /etc/nginx/sites-enabled/<subdomain>.businka.cloud
sudo nginx -t && sudo systemctl reload nginx

# Obtain certificate using webroot
sudo certbot certonly --webroot -w /var/www/html -d <subdomain>.businka.cloud

# Re-enable the subdomain config
sudo ln -s /etc/nginx/sites-available/<subdomain>.businka.cloud /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

#### Certificate renewal

Certbot automatically renews certificates via a systemd timer. To manually renew:

```bash
sudo certbot renew
```

### 3. Verify Setup

```bash
# Test Nginx configuration
sudo nginx -t

# Check certificate status
sudo certbot certificates

# Test HTTPS
curl -I https://<subdomain>.businka.cloud
```

## Deployment

Deployments are automated via GitHub Actions on push to `main` branch when files in `sites/` are modified.

The workflow:
1. Copies Nginx configs to the server via SCP
2. Places them in `/etc/nginx/sites-available/`
3. Creates symlinks in `/etc/nginx/sites-enabled/`
4. Tests and reloads Nginx

### Required GitHub Secrets

- `SERVER_HOST` - Server IP or hostname
- `SERVER_USER` - SSH username
- `SSH_PRIVATE_KEY` - SSH private key for authentication

## File Locations on Server

| Item | Path |
|------|------|
| Nginx configs | `/etc/nginx/sites-available/` |
| Enabled sites | `/etc/nginx/sites-enabled/` |
| SSL certificates | `/etc/letsencrypt/live/<domain>/` |
| Main site root | `/var/www/html` |
| Game site root | `/var/www/game` |
| Certbot logs | `/var/log/letsencrypt/` |
