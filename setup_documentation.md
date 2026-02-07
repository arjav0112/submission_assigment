# n8n Docker Setup Documentation

## Overview

This documentation covers the complete setup of n8n (workflow automation tool) running in Docker with PostgreSQL database, local Ollama AI model integration, and Cloudflare tunnel for external service connectivity.

## Architecture

```
┌─────────────────────────────────────────────┐
│           Cloudflare Tunnel                 │
│     (yourname.yourdomain.com)              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│         Docker Compose Stack                │
│                                             │
│  ┌──────────────┐  ┌──────────────┐        │
│  │     n8n      │  │  PostgreSQL  │        │
│  │   (latest)   │◄─┤   Database   │        │
│  └──────┬───────┘  └──────────────┘        │
│         │                                   │
│         ▼                                   │
│  ┌──────────────┐                          │
│  │    Ollama    │                          │
│  │ (AI Models)  │                          │
│  └──────────────┘                          │
└─────────────────────────────────────────────┘
```

## Prerequisites

- Docker and Docker Compose installed
- Domain name configured with Cloudflare
- Cloudflare account with API access
- Basic understanding of command line operations

## Directory Structure

Create the following directory structure for your n8n deployment:

```
n8n-docker/
├── docker-compose.yml
├── .env
├── data/
│   ├── n8n/
│   ├── postgres/
│   └── ollama/
└── cloudflared/
    └── config.yml
```

## Configuration Files

### 1. docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=${TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - ./data/n8n:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - n8n-network

  ollama:
    image: ollama/ollama:latest
    container_name: n8n-ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./data/ollama:/root/.ollama
    networks:
      - n8n-network

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: n8n-cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - n8n-network
    depends_on:
      - n8n

networks:
  n8n-network:
    driver: bridge
```

### 2. .env File

Create a `.env` file in the same directory with the following configuration:

```env
# PostgreSQL Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your_secure_postgres_password_here
POSTGRES_DB=n8n

# n8n Configuration
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_n8n_password_here
N8N_HOST=yoursubdomain.yourdomain.com
N8N_ENCRYPTION_KEY=your_encryption_key_here_min_10_chars

# Timezone
TIMEZONE=America/New_York

# Cloudflare Tunnel Token
CLOUDFLARE_TUNNEL_TOKEN=your_cloudflare_tunnel_token_here
```

**Important Security Notes:**
- Replace all placeholder values with strong, unique passwords
- Keep the `.env` file secure and never commit it to version control
- Add `.env` to your `.gitignore` file

## Setup Instructions

### Step 1: Create Directory Structure

```bash
mkdir -p n8n-docker/data/{n8n,postgres,ollama}
cd n8n-docker
```

### Step 2: Create Configuration Files

1. Create the `docker-compose.yml` file with the content above
2. Create the `.env` file with your actual credentials
3. Generate a secure encryption key:

```bash
# Generate encryption key (Linux/Mac)
openssl rand -base64 32

# Or use this command
head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32
```

### Step 3: Configure Cloudflare Tunnel

#### Option A: Using Cloudflare Dashboard (Recommended)

1. Log in to your Cloudflare dashboard
2. Navigate to **Zero Trust** → **Access** → **Tunnels**
3. Click **Create a tunnel**
4. Name your tunnel (e.g., "n8n-automation")
5. Copy the tunnel token provided
6. Add the token to your `.env` file as `CLOUDFLARE_TUNNEL_TOKEN`
7. Configure the tunnel route:
   - **Public hostname**: `yoursubdomain.yourdomain.com`
   - **Service**: `http://n8n:5678`

#### Option B: Using cloudflared CLI

```bash
# Install cloudflared
# For Ubuntu/Debian
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Login to Cloudflare
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create n8n-tunnel

# Get tunnel token
cloudflared tunnel token n8n-tunnel
```

### Step 4: Start the Stack

```bash
# Pull the latest images
docker-compose pull

# Start all services
docker-compose up -d

# Check logs
docker-compose logs -f
```

### Step 5: Verify Services

```bash
# Check running containers
docker-compose ps

# Should show all services as "Up"
```

### Step 6: Install Ollama Models

```bash
# Access the Ollama container
docker exec -it n8n-ollama ollama pull llama2

# Or pull other models
docker exec -it n8n-ollama ollama pull codellama
docker exec -it n8n-ollama ollama pull mistral

# List installed models
docker exec -it n8n-ollama ollama list
```

### Step 7: Access n8n

1. Open your browser and navigate to `https://yoursubdomain.yourdomain.com`
2. Log in with the credentials from your `.env` file
3. Complete the initial setup wizard


## Maintenance

### Backup Data

```bash
# Backup PostgreSQL database
docker exec n8n-postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d).sql

# Backup n8n data directory
tar -czf n8n_data_backup_$(date +%Y%m%d).tar.gz data/n8n/
```

### Update Services

```bash
# Pull latest images
docker-compose pull

# Recreate containers with new images
docker-compose up -d

# Remove old images
docker image prune -a
```

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f n8n
docker-compose logs -f postgres
docker-compose logs -f ollama
```

### Restart Services

```bash
# Restart all
docker-compose restart

# Restart specific service
docker-compose restart n8n
```

## Troubleshooting

### n8n Cannot Connect to PostgreSQL

**Symptoms:** n8n container restarts repeatedly

**Solution:**
1. Check PostgreSQL is healthy: `docker-compose ps`
2. Verify credentials in `.env` file
3. Check logs: `docker-compose logs postgres`

### Cloudflare Tunnel Not Working

**Symptoms:** Cannot access n8n via domain

**Solution:**
1. Verify tunnel token is correct
2. Check cloudflared logs: `docker-compose logs cloudflared`
3. Ensure DNS record is created in Cloudflare dashboard
4. Verify tunnel status in Cloudflare Zero Trust dashboard

### OAuth Redirects Failing

**Symptoms:** OAuth flows return to localhost instead of domain

**Solution:**
1. Ensure `N8N_HOST` is set correctly in `.env`
2. Verify `WEBHOOK_URL` includes `https://`
3. Check that the redirect URL in the external service matches your Cloudflare domain
4. Restart n8n: `docker-compose restart n8n`

### Ollama Model Not Responding

**Symptoms:** HTTP requests to Ollama timeout

**Solution:**
1. Verify model is installed: `docker exec -it n8n-ollama ollama list`
2. Test Ollama directly: `docker exec -it n8n-ollama ollama run llama2 "Hello"`
3. Check Ollama logs: `docker-compose logs ollama`
4. Ensure enough system resources (RAM/GPU)

### Database Disk Space Issues

**Symptoms:** PostgreSQL crashes or becomes unresponsive

**Solution:**
```bash
# Check disk usage
df -h

# Clean old workflow executions in n8n UI
# Settings → Executions → Delete old executions

# Vacuum PostgreSQL database
docker exec n8n-postgres vacuumdb -U n8n -d n8n -f
```

## Security Best Practices

1. **Use Strong Passwords**: Generate random passwords for all services
2. **Enable 2FA**: If n8n supports it, enable two-factor authentication
3. **Regular Backups**: Automate daily backups of database and workflows
4. **Update Regularly**: Keep Docker images updated
5. **Limit Exposure**: Use Cloudflare Access policies to restrict who can access n8n
6. **Monitor Logs**: Regularly check logs for suspicious activity
7. **Encryption Key**: Store `N8N_ENCRYPTION_KEY` securely; losing it means losing encrypted credentials

## Performance Optimization

### For Ollama

If using GPU acceleration:

```yaml
ollama:
  image: ollama/ollama:latest
  container_name: n8n-ollama
  restart: unless-stopped
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
  volumes:
    - ./data/ollama:/root/.ollama
  networks:
    - n8n-network
```

### For PostgreSQL

Add performance tuning to the postgres service:

```yaml
postgres:
  # ... existing config ...
  command: postgres -c shared_buffers=256MB -c max_connections=200
```

## Additional Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Ollama Documentation](https://ollama.ai/docs)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## Support

For issues specific to:
- **n8n**: Visit [n8n Community Forum](https://community.n8n.io/)
- **Cloudflare**: Check [Cloudflare Community](https://community.cloudflare.com/)
- **Ollama**: See [Ollama GitHub Issues](https://github.com/ollama/ollama/issues)

---

**Last Updated**: February 2026  
**Version**: 1.0  
**Maintained By**: Your Name/Team
