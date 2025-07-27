# Dokploy Deployment Guide for Mastodon

## Prerequisites

- Dokploy instance running
- Domain configured (mastodon.evansims.com)
- SSL certificates configured in Dokploy

## Deployment Steps

### 1. Create New Application in Dokploy

1. Go to your Dokploy dashboard
2. Create a new application
3. Select "Docker Compose" as the deployment type
4. Connect your Git repository

### 2. Environment Variables

In Dokploy's environment variables section, add all variables from `.env.production.example`:

**Critical Variables to Set:**

- `LOCAL_DOMAIN=evansims.com`
- `WEB_DOMAIN=mastodon.evansims.com`
- `SECRET_KEY_BASE` - Generate with: `openssl rand -base64 64`
- `OTP_SECRET` - Generate with: `openssl rand -base64 64`
- `VAPID_PRIVATE_KEY` & `VAPID_PUBLIC_KEY` - See generation instructions below
- `ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY` - See generation instructions below
- `ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT` - See generation instructions below
- `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY` - See generation instructions below
- `DB_HOST=db`
- `DB_USER=postgres`
- `DB_NAME=postgres`
- `DB_PASS` - Leave empty (trust authentication)
- `DB_PORT=5432`
- `REDIS_HOST=redis`
- `REDIS_PORT=6379`
- `SINGLE_USER_MODE=false`
- `RAILS_ENV=production`
- `NODE_ENV=production`
- SMTP credentials for your email provider

**Generating Required Keys:**

1. First, deploy with minimal config to get containers running
2. Generate encryption keys:

   ```bash
   docker exec -it [web_container_name] bin/rails db:encryption:init
   ```

   Copy the three encryption keys to Dokploy environment variables

3. Generate VAPID keys:

   ```bash
   docker exec -it [web_container_name] bundle exec rake mastodon:webpush:generate_vapid_key
   ```

   Copy both keys to Dokploy environment variables

4. Redeploy to apply all environment variables

**Important:** Dokploy will inject these as environment variables into all containers.

### 3. Volume Configuration

The docker-compose.yml uses named volumes for persistence:

- `postgres_data` - Database storage
- `redis_data` - Redis cache
- `public_system` - User uploads

These will be managed by Dokploy automatically.

### 4. Port Configuration

- Web interface: Port 3000
- Streaming API: Port 4000

Configure your Dokploy proxy/reverse proxy to:

- Route HTTPS traffic to port 3000
- Route `/api/v1/streaming` to port 4000

### 5. Initial Setup Commands

The docker-compose includes an **admin** container specifically for running maintenance commands. This container stays running and won't crash even if the database isn't initialized.

1. In Dokploy's dashboard, go to your Mastodon application
2. Find the **admin** container
3. Click on "Terminal" to open a shell
4. Run these commands:

```bash
# Initialize the database (first time only)
bundle exec rake db:setup

# Create your admin account
bin/tootctl accounts create hello --email hello@evansims.com --confirmed --skip-sign-in-token --role Owner

# Create additional accounts as needed
bin/tootctl accounts create photos --email photos@evansims.com --confirmed --skip-sign-in-token --role Moderator
```

After running `db:setup`, your web container should start successfully.

**Using the admin container for other tasks:**

- Database migrations: `bundle exec rake db:migrate`
- Clear cache: `bin/tootctl cache clear`
- Media cleanup: `bin/tootctl media remove`
- Account management: `bin/tootctl accounts`

### 6. Domain Configuration

Ensure Dokploy is configured to:

- Serve the app at https://mastodon.evansims.com
- Handle SSL termination
- Pass proper headers (X-Forwarded-For, X-Forwarded-Proto)

## Dokploy-Specific Notes

1. **No nginx container needed** - Dokploy handles reverse proxy
2. **Volumes are managed** - Don't use bind mounts
3. **Ports are simplified** - Removed localhost binding
4. **Environment variables** - Managed through Dokploy UI

## Maintenance

### Viewing Logs

Use Dokploy's log viewer or:

```bash
# Find container name first
docker ps | grep mastodon
# Then view logs
docker logs -f [container-name]
```

### Updating Mastodon

1. Update the image version in docker-compose.yml
2. Push to your repository
3. Trigger deployment in Dokploy
4. Run migrations if needed:
   ```bash
   # Find your web container name
   docker ps | grep mastodon.*web
   # Run migrations
   docker exec -it [web-container-name] bundle exec rake db:migrate
   ```

### Backup

Configure Dokploy's backup feature to regularly backup the volumes:

- postgres_data (critical)
- public_system (user uploads)
- redis_data (optional, just cache)
