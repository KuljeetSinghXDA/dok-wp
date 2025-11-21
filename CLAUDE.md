# CLAUDE.md - AI Assistant Guide for dok-wp

## Project Overview

This repository contains a production-ready Docker-based WordPress deployment optimized for the [Dokploy](https://dokploy.com/) platform. It implements a 3-tier architecture with Nginx as a reverse proxy, WordPress with PHP-FPM, and MariaDB as the database backend.

### Key Technologies

- **Web Server**: Nginx 1.28.0 (Alpine)
- **Application**: WordPress with PHP 8.4 FPM (Alpine)
- **Database**: MariaDB 11.8.3
- **Orchestration**: Docker Compose 3.8
- **Reverse Proxy**: Traefik (external, configured via labels)
- **Network**: dokploy-network (external)

## Repository Structure

```
dok-wp/
├── docker-compose.yml    # Main service orchestration
├── nginx.conf           # Nginx web server configuration
├── uploads.ini          # PHP upload and memory limits
├── low-memory.cnf       # MariaDB optimizations for low-resource environments
└── CLAUDE.md           # This file
```

### File Purposes

#### docker-compose.yml:1
The core orchestration file defining three services:

1. **nginx**: Public-facing web server
   - Serves static files with 1-month caching
   - Proxies PHP requests to WordPress container
   - Configured with Traefik labels for routing and SSL

2. **wordpress**: PHP-FPM application server
   - Runs WordPress with PHP 8.4
   - Auto-updates disabled for declarative state management
   - Mounts custom PHP configuration

3. **mariadb**: Database server
   - UTF-8 MB4 character set for full Unicode support
   - Low-memory optimizations applied
   - Persistent data storage via named volumes

#### nginx.conf:1
Nginx configuration optimized for WordPress:
- Static file caching (CSS, JS, images, fonts)
- PHP-FPM proxy to `wordpress:9000`
- Security: Denies access to `.ht*` files
- WordPress permalink support via `try_files`

#### uploads.ini:1
PHP runtime configuration:
- 64M memory limit
- 32M max upload/post size
- 300s execution timeout

#### low-memory.cnf:1
MariaDB tuning for resource-constrained environments:
- 64M InnoDB buffer pool
- Performance schema disabled
- 20 max connections
- Query cache disabled (deprecated in modern MariaDB)

## Environment Variables

The deployment requires these environment variables (typically set in Dokploy):

### Required Variables

```bash
# Domain Configuration
WP_DOMAIN=example.com

# Database Configuration
DB_NAME=wordpress
DB_USER=wp_user
DB_PASSWORD=<secure_password>
DB_ROOT_PASSWORD=<secure_root_password>
```

**Security Note**: Never commit `.env` files or hardcode credentials in docker-compose.yml.

## Development Workflows

### Making Configuration Changes

1. **Modifying Services**: Edit `docker-compose.yml`
   - Follow existing indentation (2 spaces)
   - Maintain alphabetical order in environment variables
   - Use read-only mounts (`:ro`) for configuration files

2. **Nginx Changes**: Edit `nginx.conf`
   - Test configuration before committing: `nginx -t` in container
   - Maintain security headers and caching policies

3. **PHP Settings**: Edit `uploads.ini`
   - Keep memory limits reasonable (64M default)
   - Balance upload size with available resources

4. **Database Tuning**: Edit `low-memory.cnf`
   - Adjust buffer pool size based on available RAM
   - Keep max_connections conservative (20 default)

### Git Workflow

This repository uses a simplified git workflow:

1. **Branch Naming**:
   - Claude branches use format: `claude/claude-md-<session-id>`
   - Feature branches should be descriptive: `feature/add-redis-cache`

2. **Commit Messages**:
   - Keep commits atomic and focused
   - Use present tense: "Update nginx.conf" not "Updated nginx.conf"
   - Reference issues when applicable

3. **Pushing Changes**:
   ```bash
   git add .
   git commit -m "Descriptive message"
   git push -u origin <branch-name>
   ```

### Deployment Process

This repository is designed for deployment on Dokploy:

1. Dokploy clones the repository
2. Environment variables are injected from Dokploy settings
3. `docker-compose up` is executed
4. Traefik handles SSL certificate generation via Let's Encrypt
5. Services become available at configured domain

**No manual deployment steps required** - Dokploy handles the entire lifecycle.

## Key Conventions and Best Practices

### Docker Conventions

1. **Image Selection**:
   - Prefer Alpine-based images for minimal footprint
   - Pin specific versions (not `latest`) for reproducibility
   - Example: `nginx:1.28.0-alpine`, not `nginx:alpine`

2. **Volume Mounts**:
   - Use named volumes for persistent data
   - Mount configs as read-only (`:ro`) when possible
   - Share `wordpress_data` between nginx and WordPress

3. **Networking**:
   - All services connect to external `dokploy-network`
   - Internal service communication via service names
   - Only nginx exposes ports (via Traefik)

4. **Container Naming**:
   - Use descriptive names: `wordpress_nginx`, `wordpress_app`, `wordpress_db`
   - Prefix with project name to avoid conflicts

### WordPress-Specific Conventions

1. **Auto-Updates**: Disabled to maintain infrastructure-as-code principles
   ```yaml
   WP_AUTO_UPDATE_CORE: false
   AUTOMATIC_UPDATER_DISABLED: true
   ```

2. **Table Prefix**: Standard `wp_` prefix used

3. **File Uploads**: Limited to 32M for security and performance

4. **Character Set**: UTF-8 MB4 for emoji and international character support

### Security Practices

1. **Configuration**:
   - All sensitive configs mounted as read-only
   - Nginx denies access to hidden files (`.ht*`)
   - Database not exposed externally

2. **Updates**:
   - Manual update process ensures testing
   - Pin specific image versions
   - Review changelogs before updating

3. **Passwords**:
   - Use strong, randomly generated passwords
   - Store in Dokploy environment variables
   - Never commit to git

### Resource Optimization

This setup is optimized for low-resource environments:

1. **Memory Targets**:
   - Nginx: ~10-20MB
   - WordPress/PHP-FPM: ~50-100MB
   - MariaDB: ~80-150MB
   - **Total**: ~150-300MB under normal load

2. **Scaling Considerations**:
   - Increase MariaDB `innodb_buffer_pool_size` if RAM available
   - Raise `max_connections` for higher traffic
   - Consider Redis/Memcached for object caching at scale

## Common Tasks for AI Assistants

### Adding a New Service

Example: Adding Redis for object caching

```yaml
redis:
  image: redis:7-alpine
  container_name: wordpress_redis
  restart: unless-stopped
  networks:
    - dokploy-network
  volumes:
    - redis_data:/data
```

Then add volume:
```yaml
volumes:
  redis_data:
```

### Modifying PHP Settings

Edit `uploads.ini` and adjust values:
```ini
memory_limit = 128M        # Increased from 64M
upload_max_filesize = 64M  # Increased from 32M
```

### Updating Image Versions

1. Check for new versions on Docker Hub
2. Update in docker-compose.yml:
   ```yaml
   wordpress:
     image: wordpress:php8.4-fpm-alpine  # Update version
   ```
3. Test locally if possible before committing
4. Commit with clear message: "Update WordPress to PHP 8.4"

### Adding Nginx Modules or Config

To add security headers:
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
```

Add in the `server` block in nginx.conf.

### Troubleshooting

**Database Connection Issues**:
1. Verify environment variables in Dokploy
2. Check `WORDPRESS_DB_HOST` matches service name (`mariadb:3306`)
3. Ensure database container is healthy: `docker logs wordpress_db`

**File Upload Failures**:
1. Check `uploads.ini` limits
2. Verify volume permissions
3. Review nginx client_max_body_size if needed

**Slow Performance**:
1. Review MariaDB slow query log
2. Check `innodb_buffer_pool_size` vs available RAM
3. Consider enabling WordPress object caching

## Testing Locally

To test changes locally before deployment:

```bash
# 1. Create .env file with test values
cat > .env << EOF
WP_DOMAIN=localhost
DB_NAME=wordpress
DB_USER=wp_user
DB_PASSWORD=test_password
DB_ROOT_PASSWORD=test_root_password
EOF

# 2. Create external network (if not exists)
docker network create dokploy-network

# 3. Start services
docker-compose up -d

# 4. Check logs
docker-compose logs -f

# 5. Access at http://localhost

# 6. Cleanup
docker-compose down -v
```

**Note**: Local testing won't include Traefik routing.

## Architecture Decisions

### Why Alpine Images?

- Minimal attack surface (5-10MB base vs 100MB+ for Debian)
- Faster deployment and updates
- Lower resource consumption

### Why Separate Nginx and PHP-FPM?

- Better resource isolation
- Independent scaling
- Nginx handles static files efficiently
- Standard production WordPress architecture

### Why MariaDB vs MySQL?

- Better performance in containerized environments
- Open-source governance
- Drop-in MySQL replacement with better defaults

### Why External Network?

- Dokploy manages the network lifecycle
- Allows communication with Traefik
- Enables multi-project networking if needed

## Links and Resources

- [Dokploy Documentation](https://docs.dokploy.com/)
- [WordPress Docker Images](https://hub.docker.com/_/wordpress)
- [Nginx Configuration Reference](https://nginx.org/en/docs/)
- [MariaDB Server Documentation](https://mariadb.org/documentation/)

## Change Log

### 2025-11-21
- Initial CLAUDE.md created
- Documented existing architecture and conventions
- Added comprehensive guide for AI assistants

---

**Last Updated**: 2025-11-21
**Maintained By**: AI Assistant (Claude)
**Repository**: KuljeetSinghXDA/dok-wp
