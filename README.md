# Pi-hole + Tailscale DNS Server

Network-wide ad blocking accessible via Tailscale VPN using Docker Compose.

## Features

- **Ad Blocking**: Network-wide ad and tracker blocking via Pi-hole
- **Tailscale Integration**: Secure access from any device on your Tailscale network
- **Optimized Performance**: 25,000 entry DNS cache with optimistic caching
- **Privacy Focused**: Quad9 upstream DNS with query logging
- **Auto-restart**: Containers restart automatically on failure
- **Health Checks**: Built-in monitoring for both services

## Prerequisites

- Docker and Docker Compose installed (or Coolify for managed deployments)
- Tailscale account ([sign up free](https://login.tailscale.com/start))
- Server/VPS with at least 512MB RAM
- Git (for cloning repository)
- Python 3.8+ (optional, for pre-commit hooks)

## Deployment Options

This guide covers two deployment methods:
1. **[Coolify Deployment](#coolify-deployment)** - Recommended for managed, easy deployments
2. **[Manual Docker Compose](#manual-deployment)** - Direct Docker Compose deployment

---

## Coolify Deployment

[Coolify](https://coolify.io) is an open-source, self-hosted platform that simplifies application deployment and management. It's the recommended method for deploying this Pi-hole + Tailscale setup.

### Why Coolify?

- Built-in environment variable management
- Automatic container health monitoring
- Easy updates and rollbacks
- Web-based management interface
- Persistent volume management
- No manual Docker commands needed

### Coolify Prerequisites

- Coolify installed and running ([installation guide](https://coolify.io/docs/installation))
- Git repository with this configuration
- Tailscale auth key ready

### Step-by-Step Coolify Deployment

#### 1. Create New Resource

1. Log into your Coolify dashboard
2. Click **+ Add Resource**
3. Select **Docker Compose**
4. Choose **Public Repository** or connect your Git provider

#### 2. Configure Repository

- **Repository URL**: Your Git repository URL (or use public repo)
- **Branch**: `main` (or your default branch)
- **Docker Compose Location**: Leave as `docker-compose.yml`
- **Base Directory**: Leave empty (unless compose file is in subdirectory)

#### 3. Configure Environment Variables

Coolify will automatically detect environment variables from your compose file. You need to set these in the Coolify UI:

**Required Variables:**

```
TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
PIHOLE_PASSWORD=your_strong_password_here
TZ=America/New_York
```

**Optional Variables (use defaults or customize):**

```
DNS_UPSTREAMS=9.9.9.9;149.112.112.112
CACHE_SIZE=25000
MAX_CACHE_TIME=3600
CACHE_OPTIMISTIC=true
RATE_LIMIT=1000/60
MAX_CONCURRENT=150
QUERY_LOGGING=true
DB_RETENTION_DAYS=365
DB_INTERVAL=1.0
ENABLE_IPV6=true
VIRTUAL_HOST=pihole.local
```

To add these:
1. In your Coolify resource, go to **Environment Variables** tab
2. Click **+ Add**
3. Add each variable name and value
4. For sensitive values (like `PIHOLE_PASSWORD`), toggle the "secret" option

#### 4. Important Coolify-Specific Notes

**Port Configuration:**
- **DO NOT** add port mappings in Coolify's port configuration
- Ports are defined in `docker-compose.yml` and Coolify will handle them automatically
- Adding duplicate ports will cause deployment failures

**Volume Persistence:**
- Coolify automatically creates and manages volumes defined in compose file
- Volumes persist across container updates and restarts
- Location: `/var/lib/docker/volumes/`

**Network Mode Compatibility:**
- The `network_mode: service:tailscale` configuration is supported
- Coolify will respect the network namespace sharing
- Both containers will start in correct order due to `depends_on`

**Capabilities:**
- `cap_add: NET_ADMIN` and `SYS_MODULE` are supported in Coolify
- These are necessary for Tailscale VPN functionality
- No additional Coolify configuration needed

#### 5. Generate Tailscale Auth Key

Before deploying, generate your Tailscale auth key:

1. Go to https://login.tailscale.com/admin/settings/keys
2. Click **Generate auth key**
3. Configure:
   - **Reusable**: [x] Yes (allows container recreations)
   - **Ephemeral**: [ ] No (keeps node in admin console)
   - **Tags**: `tag:dns` (optional, for ACL management)
4. Copy the key and add to Coolify environment variables as `TS_AUTHKEY`

#### 6. Deploy Application

1. Click **Deploy** button in Coolify
2. Monitor the deployment logs in real-time
3. Wait for both containers to start (typically 30-60 seconds)
4. Check deployment status shows "Running"

#### 7. Get Pi-hole's Tailscale IP

After successful deployment, retrieve the Tailscale IP:

**Option A: Using Coolify Terminal**
1. Go to your resource in Coolify
2. Click on **tailscale-pihole** container
3. Open **Terminal** tab
4. Run: `tailscale ip -4`
5. Note the IP (e.g., `100.64.1.5`)

**Option B: Using Tailscale Admin Console**
1. Go to https://login.tailscale.com/admin/machines
2. Find machine named "pihole"
3. Note the Tailscale IP address

#### 8. Configure Tailscale DNS

1. Go to https://login.tailscale.com/admin/dns
2. Enable **MagicDNS**
3. Under **Nameservers**:
   - Click **Add nameserver**
   - Select **Custom**
   - Enter your Pi-hole's Tailscale IP (e.g., `100.64.1.5`)
4. Enable **Override local DNS**
5. Click **Save**

#### 9. Configure Pi-hole Interface Settings

1. Open Pi-hole web interface: `http://<tailscale-ip>/admin`
2. Login with your `PIHOLE_PASSWORD`
3. Go to **Settings** -> **DNS**
4. Under **Interface settings**:
   - Select **Permit all origins**
   - Click **Save**

#### 10. Disable Tailscale Key Expiry

1. Go to https://login.tailscale.com/admin/machines
2. Find your "pihole" machine
3. Click **...** (three dots) -> **Disable key expiry**

### Coolify Verification

**Check Container Health:**
1. In Coolify, go to your resource
2. Both containers should show "Running" status
3. Click on **pihole** container -> **Logs** to verify no errors

**Test DNS Resolution:**
```bash
# From any device on your Tailscale network
dig @<pihole-tailscale-ip> google.com

# Test ad blocking
dig @<pihole-tailscale-ip> doubleclick.net
# Should return 0.0.0.0
```

### Coolify Updates and Maintenance

**Update Containers:**
1. Go to your resource in Coolify
2. Click **Redeploy** button
3. Coolify will pull latest images and restart containers
4. Data persists in volumes automatically

**View Logs:**
1. Click on container name (pihole or tailscale)
2. Select **Logs** tab
3. Use search and filter options

**Restart Services:**
1. Click on container name
2. Click **Restart** button
3. Or restart entire stack with **Redeploy**

### Coolify Troubleshooting

**Deployment Fails with Port Conflict:**
- Ensure you haven't added port mappings in Coolify UI
- Ports should only be in docker-compose.yml
- Check if port 53 or 80 is already in use on host

**Containers Won't Start:**
1. Check deployment logs in Coolify
2. Verify all required environment variables are set
3. Ensure Tailscale auth key is valid and not expired
4. Check container logs for specific error messages

**Can't Access Pi-hole Interface:**
1. Verify both containers are running in Coolify
2. Get Tailscale IP from container terminal
3. Ensure you're connected to Tailscale on client device
4. Try accessing via `http://<ip>/admin` (not https)

**DNS Not Working:**
1. Check Pi-hole interface setting is "Permit all origins"
2. Verify Tailscale DNS is configured in admin console
3. Ensure "Override local DNS" is enabled
4. Restart Tailscale on client device

---

## Manual Deployment

For direct Docker Compose deployment without Coolify, follow these steps:

### 1. Clone or Download

```bash
cd /path/to/your/projects
git clone <your-repo> blackhole
cd blackhole
```

### 2. Configure Environment

```bash
# Copy example environment file
cp .env.example .env

# Edit with your values
nano .env
```

**Required values in `.env`:**
- `TS_AUTHKEY`: Get from https://login.tailscale.com/admin/settings/keys
- `PIHOLE_PASSWORD`: Your Pi-hole admin password
- `TZ`: Your timezone (e.g., `America/New_York`)

### 3. Generate Tailscale Auth Key

1. Go to https://login.tailscale.com/admin/settings/keys
2. Click **Generate auth key**
3. Configure:
   - **Reusable**: [x] Yes
   - **Ephemeral**: [ ] No
   - **Tags**: `tag:dns`
4. Copy the key and paste into `.env` as `TS_AUTHKEY`

### 4. Start Services

```bash
# Create required directories
mkdir -p etc-pihole etc-dnsmasq.d tailscale-state

# Fix permissions
sudo chown -R 999:999 etc-pihole etc-dnsmasq.d

# Start containers
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f
```

### 5. Get Pi-hole's Tailscale IP

```bash
# Get the Tailscale IP address
docker exec tailscale-pihole tailscale ip -4

# Example output: 100.64.1.5
```

### 6. Configure Tailscale DNS

**Option A: Tailscale Admin Console (Recommended)**

1. Go to https://login.tailscale.com/admin/dns
2. Enable **MagicDNS**
3. Under **Nameservers**:
   - Click **Add nameserver**
   - Select **Custom**
   - Enter your Pi-hole's Tailscale IP (e.g., `100.64.1.5`)
4. Enable **Override local DNS**
5. Click **Save**

**Option B: Using Tailscale CLI**

```bash
# On any device in your tailnet
sudo tailscale up --accept-dns=true
```

### 7. Configure Pi-hole Interface Settings

1. Get Pi-hole IP: `docker exec tailscale-pihole tailscale ip -4`
2. Open Pi-hole web interface: `http://<tailscale-ip>/admin`
3. Login with your `PIHOLE_PASSWORD`
4. Go to **Settings** → **DNS**
5. Under **Interface settings**:
   - Select **Permit all origins**
   - Click **Save**

### 8. Disable Key Expiry (Important!)

1. Go to https://login.tailscale.com/admin/machines
2. Find your Pi-hole machine
3. Click **...** (three dots) -> **Disable key expiry**

## Verification

### Test DNS Resolution

```bash
# From any device on your Tailscale network
dig @<pihole-tailscale-ip> google.com
nslookup google.com <pihole-tailscale-ip>

# Test ad blocking
dig @<pihole-tailscale-ip> doubleclick.net
# Should return 0.0.0.0
```

### Check Container Health

```bash
# View logs
docker-compose logs -f pihole
docker-compose logs -f tailscale

# Check health status
docker-compose ps

# Monitor queries in real-time
docker exec pihole pihole tail
```

### Verify Tailscale Connection

```bash
# Check Tailscale status
docker exec tailscale-pihole tailscale status

# See all peers
docker exec tailscale-pihole tailscale status --peers
```

## Accessing Pi-hole

### Web Interface

- URL: `http://<tailscale-ip>/admin`
- Password: Value from `PIHOLE_PASSWORD` in `.env`

### CLI Commands

```bash
# View Pi-hole status
docker exec pihole pihole status

# View recent queries
docker exec pihole pihole -q

# Update gravity (blocklists)
docker exec pihole pihole -g

# Restart DNS service
docker exec pihole pihole restartdns

# View Pi-hole version
docker exec pihole pihole -v
```

## Maintenance

### Update Containers

```bash
# Pull latest images
docker-compose pull

# Recreate containers
docker-compose up -d

# Clean up old images
docker image prune -f
```

### Update Blocklists

Blocklists update automatically every week. To manually update:

```bash
docker exec pihole pihole -g
```

### Backup Configuration

```bash
# Backup Pi-hole settings
docker exec pihole pihole -a -t

# Or backup the entire directory
tar -czf pihole-backup-$(date +%Y%m%d).tar.gz etc-pihole etc-dnsmasq.d tailscale-state
```

### View Logs

```bash
# All logs
docker-compose logs

# Follow logs
docker-compose logs -f

# Specific service
docker-compose logs pihole
docker-compose logs tailscale
```

## Troubleshooting

### DNS Queries Not Appearing in Pi-hole

1. Check interface setting is "Permit all origins"
2. Verify Tailscale DNS is configured: https://login.tailscale.com/admin/dns
3. Check "Override local DNS" is enabled
4. Test direct query: `dig @<pihole-ip> google.com`

### Can't Access Web Interface

```bash
# Get current Tailscale IP
docker exec tailscale-pihole tailscale ip -4

# Check if Pi-hole container is running
docker-compose ps

# Check logs for errors
docker-compose logs pihole
```

### Container Keeps Restarting

```bash
# Check logs for errors
docker-compose logs pihole

# Common issue: Port 53 in use
sudo lsof -i :53
# If found, stop conflicting service

# Check permissions
sudo chown -R 999:999 etc-pihole etc-dnsmasq.d
```

### Tailscale Not Connecting

```bash
# Check Tailscale logs
docker-compose logs tailscale

# Verify auth key is valid
# Generate new key if expired: https://login.tailscale.com/admin/settings/keys

# Restart Tailscale container
docker-compose restart tailscale
```

### DNS Not Working on Devices

1. Check Tailscale DNS settings: https://login.tailscale.com/admin/dns
2. Ensure "Override local DNS" is enabled
3. On device, restart Tailscale:
   ```bash
   # Linux/Mac
   sudo tailscale down && sudo tailscale up

   # Windows: Restart Tailscale service
   ```

### No Internet Connection

```bash
# Check upstream DNS is configured
docker exec pihole cat /etc/pihole/setupVars.conf | grep DNS

# Test upstream DNS
docker exec pihole dig @9.9.9.9 google.com

# If fails, update DNS_UPSTREAMS in .env and restart
docker-compose restart pihole
```

## Advanced Configuration

### Custom DNS Settings

Create custom dnsmasq config files in `etc-dnsmasq.d/`:

```bash
# Example: Local DNS records
cat > etc-dnsmasq.d/04-custom.conf << EOF
# Local domain resolution
address=/mydevice.local/192.168.1.100

# Custom DNS for specific domain
server=/example.com/1.1.1.1
EOF

# Restart to apply
docker-compose restart pihole
```

### Add Custom Blocklists

1. Access Pi-hole web interface
2. Go to **Group Management** → **Adlists**
3. Add blocklist URL
4. Update gravity: `docker exec pihole pihole -g`

**Recommended blocklists:**
- StevenBlack's Unified: https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
- EasyList: https://v.firebog.net/hosts/Easylist.txt

### Firewall Rules (Optional)

If you want to restrict access further:

```bash
# Allow DNS only from Tailscale
sudo ufw allow in on tailscale0 to any port 53
sudo ufw allow in on tailscale0 to any port 80

# Deny from other interfaces (if not needed)
sudo ufw deny 53/tcp
sudo ufw deny 53/udp
```

### Resource Adjustment

Edit `docker-compose.yml` under `deploy.resources` section:

```yaml
deploy:
  resources:
    limits:
      cpus: '2'        # Adjust as needed
      memory: 512M     # Adjust as needed
    reservations:
      cpus: '0.5'
      memory: 256M
```

## Uninstalling

```bash
# Stop and remove containers
docker-compose down

# Remove all data (optional)
rm -rf etc-pihole etc-dnsmasq.d tailscale-state

# Remove from Tailscale admin console
# https://login.tailscale.com/admin/machines
```

## Security

This infrastructure implements multiple security layers to protect your DNS and network.

### Security Features

- **Container Hardening**: Minimal capabilities (NET_ADMIN only), no privilege escalation
- **Resource Limits**: CPU, memory, PID, and swap limits prevent resource exhaustion
- **Network Isolation**: Access restricted to Tailscale VPN only
- **Rate Limiting**: DNS queries limited to 100/minute to prevent amplification attacks
- **DNSSEC Enabled**: Protects against DNS spoofing and cache poisoning
- **Ephemeral Keys**: Tailscale auth keys automatically expire for better security
- **Privacy Protection**: Query logs retained for only 7 days (configurable)

### Critical Security Requirements

1. **NEVER expose ports 53 or 80 to the public internet**
   - Only access via Tailscale VPN
   - Verify firewall rules block public access

2. **Use strong passwords** (minimum 20 characters)
   ```bash
   # Generate secure password
   openssl rand -base64 32
   ```

3. **Use ephemeral Tailscale keys**
   - Set "Ephemeral: Yes" when generating auth keys
   - Automatically handled by docker-compose.yml

4. **Disable key expiry after deployment**
   - Prevents service interruption
   - Done via Tailscale admin console

5. **Regular updates**
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

6. **Monitor for suspicious activity**
   ```bash
   docker-compose logs -f pihole
   docker exec pihole pihole -t
   ```

## Development

### Pre-Commit Hooks

This repository uses pre-commit hooks to ensure code quality and prevent common issues.

**Setup (one-time):**
```bash
# Install pre-commit
pip install pre-commit

# Install git hooks
pre-commit install

# (Optional) Run on all files
pre-commit run --all-files
```

**What gets checked:**
- YAML syntax and formatting
- Docker Compose file validation
- Markdown linting
- Secret detection (prevents committing sensitive data)
- Trailing whitespace and line endings
- .env file synchronization with .env.example

**Daily usage:**
Pre-commit runs automatically when you `git commit`. If checks fail, fix the issues and commit again.

**Bypass hooks (emergency only):**
```bash
git commit --no-verify -m "Emergency fix"
```

## Support

- Pi-hole Documentation: https://docs.pi-hole.net/
- Tailscale Documentation: https://tailscale.com/kb/
- Docker Compose Reference: https://docs.docker.com/compose/
- Coolify Documentation: https://coolify.io/docs

## License

This configuration is provided as-is for personal use.
