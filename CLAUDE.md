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
2. **memcached**: Alpine-based memcached for caching (1GB allocation)
3. **seafile**: Main Seafile application with volume mounts and environment configuration

### Network Configuration
- **traefik**: External network for reverse proxy routing
- **seafile-network**: Dedicated network for service communication

### Storage
- Database: `/home/server/files/seafile-mysql` mounted to `/var/lib/mysql`
- Seafile data: `/home/server/files/seafile` mounted to `/shared`
- Library data: `/mnt/nas-direct/seafile-library` mounted to `/shared/seafile/seafile-data`

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
- `SEAFILE_ADMIN_EMAIL`: Initial admin account email
- `SEAFILE_ADMIN_PASSWORD`: Initial admin account password
- `SEAFILE_SERVER_HOSTNAME`: Domain name for the service (files.waugze.com)

### Network Architecture

- **traefik**: External network for reverse proxy routing
- **seafile-network**: Dedicated network for service communication
- **Database isolation**: MariaDB and Memcached only on seafile-network
- **Public access**: Only Seafile service exposed via Traefik

### Traefik Labels
The service includes Traefik configuration for:
- HTTPS termination with Let's Encrypt
- Custom domain routing (`files.waugze.com`)
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

### Required Drone Secrets

Configure these secrets in your Drone CI dashboard:

```bash
# Seafile configuration
SEAFILE_MYSQL_ROOT_PASSWORD    # Database root password
SEAFILE_ADMIN_EMAIL           # Admin account email
SEAFILE_ADMIN_PASSWORD        # Admin account password
SEAFILE_SERVER_HOSTNAME       # Server hostname (files.waugze.com)
```

### Deployment Process

1. **Push to main branch** triggers automatic deployment
2. **Cleanup step** removes volumes and prunes Docker system
3. **Network creation** ensures dedicated network exists
4. **Service deployment** pulls latest images and starts services
5. **Health verification** confirms services are running

## Development Workflow

This is primarily a deployment configuration, not a development codebase. Changes typically involve:

1. **Environment Updates**: Modify Drone secrets for configuration changes
2. **Service Configuration**: Update `docker-compose.yml` for service modifications
3. **Deploy Changes**: Push to `main` branch to trigger automated deployment
4. **Monitor**: Use `docker-compose logs -f` to monitor service health

## Troubleshooting

### Common Issues
- **User/Permission Issues**: Fixed by running Seafile container as root (`user: "0:0"`)
- **Mount Structure Conflicts**: Use single mount point for initial setup
- **Database Connection**: Ensure MariaDB is healthy before Seafile starts
- **File Upload Limits**: Check Traefik timeout middleware configuration
- **SSL/TLS**: Verify Let's Encrypt certificate generation through Traefik
- **Data Persistence**: Ensure volume mounts are correctly configured
- **Existing Data Conflicts**: Pipeline includes cleanup step to ensure fresh deployment

### Manual Cleanup (if needed)
```bash
# Stop all containers and remove volumes
docker-compose down --volumes

# Remove seafile directory for fresh start
rm -rf /home/server/files/seafile

# Clean Docker system
docker system prune -f

# Redeploy
docker-compose up -d
```

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
- **SSL/TLS**: Automatic HTTPS via Let's Encrypt
- **Security Headers**: Applied via Traefik middleware
- **Health Checks**: Automatic service monitoring and restart