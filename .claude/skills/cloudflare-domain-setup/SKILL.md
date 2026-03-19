---
name: cloudflare-domain-setup
description: Configure Cloudflare for a domain via API. Sets up zone, DNS records, SSL Full Strict with Origin Certificate, security settings, and cache page rules. Use when user wants to set up Cloudflare for a WordPress domain.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - WebSearch
  - Agent
---

# Cloudflare Domain Setup Skill

## Purpose
Configure Cloudflare for a WordPress domain: zone creation, DNS records, SSL Full (Strict) with Origin Certificate, security settings, and page rules.

## Required Input (from user/orchestrator, NEVER save to files)
- Cloudflare email
- Cloudflare Global API Key
- Domain name
- Server IP (from vps-server-setup or user)
- SSH host, user, password, port (for installing origin cert on server)

## API Authentication
All Cloudflare API calls use:
```bash
curl -s -X METHOD "https://api.cloudflare.com/client/v4/ENDPOINT" \
  -H "X-Auth-Email: EMAIL" \
  -H "X-Auth-Key: API_KEY" \
  -H "Content-Type: application/json" \
  --data 'JSON_BODY'
```

Ensure `jq` is available: `command -v jq &>/dev/null || apt install jq -y` (or `brew install jq` on macOS).

## SSH Connection (for server-side operations)
```bash
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'COMMAND'
```

If `sshpass` not installed locally:
- macOS: `brew install hudochenkov/sshpass/sshpass`
- Linux: `apt install sshpass -y`

## Setup Checklist

### 1. Check/Create Zone
```bash
# Check existing zone
ZONE_RESPONSE=$(curl -s "https://api.cloudflare.com/client/v4/zones?name=DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
ZONE_ID=$(echo "$ZONE_RESPONSE" | jq -r '.result[0].id // empty')

# Create if not exists
if [ -z "$ZONE_ID" ]; then
  ZONE_RESPONSE=$(curl -s -X POST "https://api.cloudflare.com/client/v4/zones" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"name":"DOMAIN","jump_start":true}')
  ZONE_ID=$(echo "$ZONE_RESPONSE" | jq -r '.result.id')
fi

# Extract nameservers
NS1=$(echo "$ZONE_RESPONSE" | jq -r '.result.name_servers[0] // .result[0].name_servers[0]')
NS2=$(echo "$ZONE_RESPONSE" | jq -r '.result.name_servers[1] // .result[0].name_servers[1]')
```

### 2. DNS Records
Check existing before creating. Update if exists, create if not.

**A Record (@ → Server IP, proxied):**
```bash
# Check existing A record
EXISTING_A=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=A&name=DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
RECORD_ID=$(echo "$EXISTING_A" | jq -r '.result[0].id // empty')

if [ -n "$RECORD_ID" ]; then
  # Update existing
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${RECORD_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"A","name":"@","content":"SERVER_IP","proxied":true}'
else
  # Create new
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"A","name":"@","content":"SERVER_IP","proxied":true}'
fi
```

**www CNAME (www → domain, proxied):**
```bash
# Check existing www CNAME
EXISTING_WWW=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=CNAME&name=www.DOMAIN" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json")
WWW_ID=$(echo "$EXISTING_WWW" | jq -r '.result[0].id // empty')

if [ -n "$WWW_ID" ]; then
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${WWW_ID}" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"CNAME","name":"www","content":"DOMAIN","proxied":true}'
else
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
    -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
    --data '{"type":"CNAME","name":"www","content":"DOMAIN","proxied":true}'
fi
```

### 3. SSL Mode — Full (Strict)
```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings/ssl" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{"value":"strict"}'
```

### 4. Generate Origin Certificate
**NOTE:** The Origin CA API uses a different auth header than the zone API. Use `X-Auth-User-Service-Key` with the Global API Key (same key value, different header name). If that fails, try the Origin CA Key from the Cloudflare dashboard (API Tokens → Origin CA Key).
```bash
CERT_RESPONSE=$(curl -s -X POST "https://api.cloudflare.com/client/v4/certificates" \
  -H "X-Auth-User-Service-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "hostnames": ["DOMAIN", "*.DOMAIN"],
    "requested_validity": 5475,
    "request_type": "origin-rsa",
    "csr": ""
  }')

CERTIFICATE=$(echo "$CERT_RESPONSE" | jq -r '.result.certificate')
PRIVATE_KEY=$(echo "$CERT_RESPONSE" | jq -r '.result.private_key')
```
**IMPORTANT:** Never log or output the private key content.

### 5. Install Origin Cert on Server (via SSH)
```bash
# Create SSL directory
sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'mkdir -p /etc/ssl/cloudflare/'

# Write certificate
echo "$CERTIFICATE" | sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'cat > /etc/ssl/cloudflare/DOMAIN.pem'

# Write private key
echo "$PRIVATE_KEY" | sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST 'cat > /etc/ssl/cloudflare/DOMAIN.key'

# Set permissions
sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST '
  chmod 644 /etc/ssl/cloudflare/DOMAIN.pem
  chmod 600 /etc/ssl/cloudflare/DOMAIN.key
  chown root:root /etc/ssl/cloudflare/*
'
```

### 6. Update Nginx with SSL
Detect PHP-FPM socket first:
```bash
PHP_SOCK=$(sshpass -p 'PASS' ssh -o StrictHostKeyChecking=no -p PORT USER@HOST \
  'find /run/php -name "*.sock" 2>/dev/null | head -1')
```

Write new Nginx config replacing the HTTP-only block:
```nginx
# HTTPS server
server {
    listen 443 ssl;
    server_name DOMAIN www.DOMAIN;
    root /var/www/DOMAIN;
    index index.php index.html;

    ssl_certificate /etc/ssl/cloudflare/DOMAIN.pem;
    ssl_certificate_key /etc/ssl/cloudflare/DOMAIN.key;

    # Security: block xmlrpc
    location = /xmlrpc.php { deny all; }

    # Security: block REST API user enumeration
    location ~ ^/wp-json/wp/v2/users { deny all; }

    # Security: block wp-includes browsing
    location ~* /wp-includes/.*\.php$ { deny all; }

    # PHP handling
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:PHP_FPM_SOCKET;
    }

    # WordPress permalinks
    location / {
        try_files \$uri \$uri/ /index.php?\$args;
    }

    # Static file caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)\$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Deny hidden files
    location ~ /\. { deny all; }
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name DOMAIN www.DOMAIN;
    return 301 https://\$host\$request_uri;
}
```

Write config, test, and reload:
```bash
# Write config via SSH (use heredoc)
# nginx -t && systemctl reload nginx
```
**CRITICAL:** Always run `nginx -t` before `systemctl reload nginx`. If test fails, fix config before reloading.

### 7. Security Settings (API)
Apply each setting via PATCH (use full API URL with zone_id):
```bash
BASE="https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/settings"
AUTH='-H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json"'

# Always Use HTTPS
curl -s -X PATCH "${BASE}/always_use_https" ${AUTH} --data '{"value":"on"}'

# Minimum TLS 1.2
curl -s -X PATCH "${BASE}/min_tls_version" ${AUTH} --data '{"value":"1.2"}'

# Auto HTTPS Rewrites
curl -s -X PATCH "${BASE}/automatic_https_rewrites" ${AUTH} --data '{"value":"on"}'

# Brotli compression
curl -s -X PATCH "${BASE}/brotli" ${AUTH} --data '{"value":"on"}'

# Security level
curl -s -X PATCH "${BASE}/security_level" ${AUTH} --data '{"value":"medium"}'

# Browser integrity check
curl -s -X PATCH "${BASE}/browser_check" ${AUTH} --data '{"value":"on"}'

# HSTS
curl -s -X PATCH "${BASE}/security_header" ${AUTH} \
  --data '{"value":{"strict_transport_security":{"enabled":true,"max_age":31536000,"include_subdomains":true,"nosniff":true}}}'
```

### 8. Page Rules
Check existing rules count first (free plan = 3 max):
```bash
RULES_COUNT=$(curl -s "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" | jq '.result | length')
```

If rules available, create:
```bash
# wp-admin: bypass cache + high security
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "targets":[{"target":"url","constraint":{"operator":"matches","value":"*DOMAIN/wp-admin/*"}}],
    "actions":[{"id":"cache_level","value":"bypass"},{"id":"security_level","value":"high"}],
    "status":"active"
  }'

# wp-login: bypass cache + high security
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/pagerules" \
  -H "X-Auth-Email: EMAIL" -H "X-Auth-Key: KEY" -H "Content-Type: application/json" \
  --data '{
    "targets":[{"target":"url","constraint":{"operator":"matches","value":"*DOMAIN/wp-login.php*"}}],
    "actions":[{"id":"cache_level","value":"bypass"},{"id":"security_level","value":"high"}],
    "status":"active"
  }'
```

If page rules limit reached, warn user and skip.

## Output
Report these results to the orchestrator:
```
## Cloudflare Setup Results
- **Zone ID:** {zone_id}
- **Nameservers:** {ns1}, {ns2}
- **SSL Mode:** Full (Strict)
- **Origin Certificate:** Installed (expires ~2041)
- **DNS:** A @ → {SERVER_IP} (proxied), CNAME www → {domain} (proxied)
- **Settings Applied:** Always HTTPS, TLS 1.2+, HSTS, Brotli, Medium Security
- **Page Rules:** wp-admin bypass cache, wp-login bypass cache
```

## Manual Step Required
```
## ACTION REQUIRED — Change Nameservers
Go to your domain registrar and change nameservers to:
1. {nameserver1}
2. {nameserver2}
DNS propagation takes 24-48 hours. Site will work after propagation completes.
```

## Error Handling
- If zone already exists → reuse it (extract zone_id)
- If DNS records exist → update (PUT) instead of create (POST)
- If page rules limit reached → warn user, skip page rules
- If origin cert generation fails → check API key has zone permissions
- Always `nginx -t` before reload — if fail, do NOT reload
- If API returns 403 → API key lacks permissions, inform user
- If API returns 429 → rate limited, wait 60s and retry once

## Important Rules
- NEVER save API keys or credentials to any file
- NEVER log or output the private key content
- Origin cert private key file must be 600 permissions (owner-only)
- SSL directory owned by root
- Always test nginx config before reloading
- Use `jq` for JSON parsing
- Run all server-side commands via SSH remotely
- Every step checks existing state before acting (idempotent)
