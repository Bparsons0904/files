# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Seafile deployment project that uses Docker Compose to orchestrate a self-hosted file sharing and collaboration platform. The deployment includes:

- **Seafile**: Main file sharing service (seafileltd/seafile-mc:latest)
- **MariaDB**: Database backend for Seafile
- **Memcached**: Caching layer for improved performance
- **Traefik Integration**: Reverse proxy with automatic HTTPS via Let's Encrypt

## Architecture

The application consists of three main services:

1. **seafile-mysql**: MariaDB 10.11 database container with persistent storage
2. **memcached**: Alpine-based memcached for caching (256MB allocation)
3. **seafile**: Main Seafile application with volume mounts and environment configuration

### Network Configuration
- **seafile**: Internal bridge network for service communication
- **proxy**: External network for Traefik reverse proxy integration

### Storage
- Database: `${SEAFILE_MYSQL_DB}` directory mounted to `/var/lib/mysql`
- Seafile data: `${SEAFILE_DATA}` directory mounted to `/shared`

## Common Commands

### Environment Setup
```bash
# Copy environment template and configure
cp .env.example .env
# Edit .env with your specific values
```

### Network Setup
```bash
# Create dedicated network (required for deployment)
docker network create seafile-network

# Create NAS storage directories
sudo mkdir -p /mnt/nas-direct/seafile-data
sudo mkdir -p /mnt/nas-direct/seafile-mysql
sudo chown -R 1000:1000 /mnt/nas-direct/seafile-data
sudo chown -R 1000:1000 /mnt/nas-direct/seafile-mysql
```

### Service Management
```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f seafile
docker-compose logs -f seafile-mysql
docker-compose logs -f memcached

# Restart specific service
docker-compose restart seafile

# Pull latest images
docker-compose pull

# Rebuild and restart
docker-compose up -d --build
```

### Maintenance Commands
```bash
# Check service status
docker-compose ps

# Execute commands in seafile container
docker-compose exec seafile bash

# Database backup
docker-compose exec seafile-mysql mysqldump -u root -p seafile > backup.sql

# View container resource usage
docker stats
```

## Configuration

### Environment Variables
All configuration is managed through Drone secrets (production) or environment variables (local development):

- `SEAFILE_MYSQL_ROOT_PASSWORD`: Database root password
- `SEAFILE_MYSQL_DB`: Database volume mount path (local dev only)
- `SEAFILE_DATA`: Seafile data volume mount path (local dev only)
- `SEAFILE_ADMIN_EMAIL`: Initial admin account email
- `SEAFILE_ADMIN_PASSWORD`: Initial admin account password
- `SEAFILE_SERVER_HOSTNAME`: Domain name for the service

### Storage Configuration

**Production (NAS Storage)**:
- Database: `/mnt/nas-direct/seafile-mysql`
- Seafile data: `/mnt/nas-direct/seafile-data`
- High-performance SMB/CIFS mount (285 MB/s)

**Local Development**:
- Use `.env` file with local paths
- Database: `./seafile-mysql`
- Seafile data: `./seafile-data`

### Network Architecture

- **traefik**: External network for reverse proxy routing
- **seafile-network**: Dedicated network for service communication
- **Database isolation**: MariaDB and Memcached only on seafile-network
- **Public access**: Only Seafile service exposed via Traefik

### Traefik Labels
The service includes Traefik configuration for:
- HTTPS termination with Let's Encrypt
- Custom domain routing (`files.bobparsons.dev`)
- Security headers middleware
- Extended timeout middleware for large file uploads (3600s)
- Health checks and automatic failover

## CI/CD Pipeline

### Drone CI Configuration

This project uses Drone CI for automated deployment. The `.drone.yml` configuration includes:

- **Automated deployment** on pushes to `main` branch
- **Environment variable management** via Drone secrets
- **Health checks** to verify successful deployment
- **Cleanup tasks** to manage Docker images
- **Notifications** for deployment status

### Required Drone Secrets

Configure these secrets in your Drone CI dashboard:

```bash
# Seafile configuration
seafile_mysql_root_password    # Database root password
seafile_mysql_db              # Database volume path
seafile_data                  # Seafile data volume path
seafile_admin_email           # Admin account email
seafile_admin_password        # Admin account password
seafile_server_hostname       # Server hostname (files.bobparsons.dev)

# Notification webhooks (optional)
webhook_success_url           # Success notification webhook
webhook_failure_url           # Failure notification webhook
```

### Deployment Process

1. **Push to main branch** triggers automatic deployment
2. **Network creation** ensures dedicated network exists
3. **Storage preparation** creates NAS directories
4. **Service deployment** pulls latest images and starts services
5. **Health verification** confirms services are running
6. **Cleanup** removes old Docker images

## Development Workflow

This is primarily a deployment configuration, not a development codebase. Changes typically involve:

1. **Environment Updates**: Modify Drone secrets for configuration changes
2. **Service Configuration**: Update `docker-compose.yml` for service modifications
3. **Deploy Changes**: Push to `main` branch to trigger automated deployment
4. **Monitor**: Use `docker-compose logs -f` to monitor service health

## Troubleshooting

### Common Issues
- **Database Connection**: Ensure MariaDB is healthy before Seafile starts
- **File Upload Limits**: Check Traefik timeout middleware configuration
- **SSL/TLS**: Verify Let's Encrypt certificate generation through Traefik
- **Data Persistence**: Ensure volume mounts are correctly configured

### Health Checks
```bash
# Check if services are running
docker-compose ps

# Verify database connectivity
docker-compose exec seafile-mysql mysql -u root -p -e "SHOW DATABASES;"

# Check Seafile initialization
docker-compose exec seafile ls -la /shared
```

## Security Considerations

- **Secrets Management**: All sensitive data stored in Drone secrets
- **Network Isolation**: Database services isolated from public networks
- **File Permissions**: Proper ownership on NAS storage directories
- **SSL/TLS**: Automatic HTTPS via Let's Encrypt
- **Security Headers**: Applied via Traefik middleware
- **Resource Limits**: Memory and CPU limits prevent resource exhaustion
- **Health Checks**: Automatic service monitoring and restart
- **Regular Updates**: Automated image updates via CI/CD pipeline