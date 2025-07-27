# Mastodon Server Setup Instructions

## Security Setup (IMPORTANT)

### For Local Development:
1. Copy `.env.production.example` to `.env.production`:
   ```bash
   cp .env.production.example .env.production
   ```

2. **NEVER commit `.env.production` to version control** - it contains secrets!

3. The `.gitignore` file will prevent accidental commits of:
   - Environment files with secrets
   - Database data
   - SSL certificates
   - User-uploaded content

### For Dokploy Deployment:
- Use Dokploy's environment variable management instead of `.env.production`
- See `DOKPLOY_SETUP.md` for detailed instructions
- All secrets should be configured through Dokploy's UI

## Initial Setup

1. **Generate secrets for .env.production:**
   ```bash
   docker-compose run --rm web bundle exec rake secret
   ```
   Add the output to `SECRET_KEY_BASE` in `.env.production`

   ```bash
   docker-compose run --rm web bundle exec rake secret
   ```
   Add the output to `OTP_SECRET` in `.env.production`

2. **Generate VAPID keys:**
   ```bash
   docker-compose run --rm web bundle exec rake mastodon:webpush:generate_vapid_key
   ```
   Add the private and public keys to `VAPID_PRIVATE_KEY` and `VAPID_PUBLIC_KEY` in `.env.production`

3. **Setup the database:**
   ```bash
   docker-compose run --rm web bundle exec rake db:setup
   ```

4. **Precompile assets:**
   ```bash
   docker-compose run --rm web bundle exec rake assets:precompile
   ```

5. **Create your accounts:**
   
   Create your main admin account:
   ```bash
   docker-compose run --rm web bin/tootctl accounts create hello --email hello@evansims.com --confirmed --role Owner
   ```
   
   Create additional accounts:
   ```bash
   docker-compose run --rm web bin/tootctl accounts create photos --email photos@evansims.com --confirmed --role Moderator
   ```
   
   You can create as many accounts as you need. Each will have the format `username@evansims.com`.

## Starting the Server

```bash
docker-compose up -d
```

## SSL Certificate Setup

1. Install certbot:
   ```bash
   sudo apt-get install certbot
   ```

2. Obtain certificate:
   ```bash
   sudo certbot certonly --standalone -d mastodon.evansims.com
   ```

3. Uncomment the SSL certificate lines in `nginx.conf`

## Important Notes

- The `LOCAL_DOMAIN` is set to `evansims.com` as requested
- The `WEB_DOMAIN` is set to `mastodon.evansims.com`
- This configuration supports multiple accounts (hello@evansims.com, photos@evansims.com, etc.)
- Public registrations are disabled (`REGISTRATIONS_MODE=none`)
- Elasticsearch is disabled by default (uncomment in docker-compose.yml to enable)
- Remember to configure your SMTP settings in `.env.production`
- The nginx configuration assumes you're running it on the host machine, not in a container

## Managing Multiple Accounts

### Creating New Accounts
```bash
docker-compose run --rm web bin/tootctl accounts create USERNAME --email EMAIL@evansims.com --confirmed
```

### Account Roles
- **Owner**: Full admin access (use for your main account)
- **Admin**: Administrative access
- **Moderator**: Can moderate content
- **User**: Regular user (default)

### Reset Password for an Account
```bash
docker-compose run --rm web bin/tootctl accounts modify USERNAME --reset-password
```

### List All Accounts
```bash
docker-compose run --rm web bin/tootctl accounts list
```

### Make an Account Admin
```bash
docker-compose run --rm web bin/tootctl accounts modify USERNAME --role Admin
```

## Maintenance Commands

- View logs: `docker-compose logs -f`
- Stop services: `docker-compose down`
- Update Mastodon: Change the image version in docker-compose.yml and run `docker-compose pull && docker-compose up -d`
- Backup database: `docker-compose exec db pg_dump -U postgres postgres > backup.sql`