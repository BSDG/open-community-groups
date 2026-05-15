# OCG Quick Start Checklist

## Pre-Deployment Checklist

- [ ] Docker & Docker Compose installed
- [ ] **PostgreSQL cluster prepared**
  - [ ] Database created (`ocg` or similar)
  - [ ] User created with password
  - [ ] Permissions granted
  - [ ] Network accessible from Docker
  - [ ] Connection tested from Docker host
- [ ] Created `.env` file (copy from `.env.example`)
- [ ] Brevo account created and SMTP credentials obtained
- [ ] **GitHub OAuth app created** (optional but recommended)
  - [ ] Registered in GitHub Developer Settings
  - [ ] Client ID and Secret saved
  - [ ] Callback URL configured: `https://yourdomain.com/log-in/oauth2/github/callback`
- [ ] Reverse proxy configured and pointed to your server
- [ ] Domain DNS records updated (if using a real domain)
- [ ] SSL/TLS certificate ready for your reverse proxy

---

## Your Deployment Checklist

### What You Need to Fill In

1. **`.env` file** — Update these values:
   ```
   DB_HOST=your-postgres-cluster.example.com
   DB_PORT=5432
   DB_NAME=ocg
   DB_USER=ocg
   DB_PASSWORD=your_secure_password
   SERVER_BASE_URL=https://yourdomain        # Your public domain
   EMAIL_FROM_ADDRESS=noreply@yourdomain     # Sender email
   SMTP_USERNAME=brevo-account@email.com     # Your Brevo email
   SMTP_PASSWORD=your_brevo_smtp_key         # Brevo SMTP key
   GITHUB_CLIENT_ID=                         # (optional) From GitHub OAuth app
   GITHUB_CLIENT_SECRET=                     # (optional) From GitHub OAuth app
   ```

2. **Reverse proxy config** — Point traffic to `http://localhost:9000`
   - Nginx / Traefik / Caddy / Cloudflare Tunnel / etc.
   - Must forward `Host` and `X-Forwarded-Proto` headers
   - Enable HTTPS/SSL at the proxy level

3. **Domain setup** — Your reverse proxy should listen on your domain

---

## Deployment Steps

## Deployment Steps

```bash
# 1. Clone or navigate to the repo
cd /path/to/open-community-groups

# 2. Prepare PostgreSQL (if not done yet)
psql -h your-postgres-host.com -U postgres -d postgres
# CREATE USER ocg WITH PASSWORD 'strong_password';
# CREATE DATABASE ocg OWNER ocg;
# GRANT ALL PRIVILEGES ON DATABASE ocg TO ocg;

# 3. Copy and edit the environment file
cp .env.example .env
# Edit .env with your database, email, and GitHub credentials

# 4. Start all services
docker-compose up -d

# 5. Wait for migrations to complete (~30s)
docker-compose logs -f ocg-server

# 6. Access your app
# via reverse proxy: https://yourdomain.com
# or local testing: http://localhost:9000
```

---

## Services Running

| Service | Container | Port (internal) | What it does |
|---------|-----------|---|---|
| **PostgreSQL** | `ocg-postgres` | 5432 | Database |
| **DB Migrator** | `ocg-db-migrator` | — | Runs migrations once, then exits |
| **OCG Server** | `ocg-server` | 9000 | Main web app |

---

## Testing Email Delivery

1. Sign up with an email on the platform
2. You should receive a verification email within 60 seconds
3. Check spam folder if not received
4. Verify in Brevo dashboard that email was sent

---

## Enabling GitHub OAuth (Optional)

If you created a GitHub OAuth app:

1. Set in `.env`:
   ```
   GITHUB_OAUTH_ENABLED=true
   GITHUB_CLIENT_ID=xxx
   GITHUB_CLIENT_SECRET=xxx
   ```

2. Restart the server:
   ```bash
   docker-compose restart ocg-server
   ```

3. You should now see "Log in with GitHub" option

---

## Monitoring & Maintenance

### View Logs
```bash
docker-compose logs -f ocg-server          # Follow server logs
docker-compose logs db-migrator            # Check migration logs
docker-compose logs postgres               # Database logs
```

### Check Status
```bash
docker-compose ps                          # See all containers
docker-compose exec postgres psql -U ocg   # Connect to database
```

### Restart/Stop
```bash
docker-compose restart ocg-server          # Restart just the server
docker-compose down                        # Stop everything
docker-compose up -d                       # Start again
```

### Update/Rebuild
```bash
git pull origin main                       # Get latest code
docker-compose down                        # Stop
docker-compose build --no-cache            # Rebuild images
docker-compose up -d                       # Start fresh
```

---

## Backup Strategy

PostgreSQL data is in the `postgres_data` volume. To backup:

```bash
# Backup database
docker-compose exec postgres pg_dump -U ocg -d ocg > backup.sql

# Backup volume (macOS/Linux)
docker run --rm -v postgres_data:/data -v $PWD:/backup \
  alpine tar czf /backup/postgres_backup.tar.gz -C /data .
```

---

## Production Considerations

- [ ] Use strong, random `DB_PASSWORD` (min 32 chars)
- [ ] Set `COOKIE_SECURE=true` (requires HTTPS)
- [ ] Use a managed database instead of in-container PostgreSQL
- [ ] Configure log aggregation (ELK, Datadog, etc.)
- [ ] Set up automated backups
- [ ] Use a reverse proxy with SSL termination
- [ ] Consider using S3/object storage for image uploads
- [ ] Monitor email delivery rates via Brevo dashboard
- [ ] Set up uptime monitoring for your domain
- [ ] Plan for database scaling as you grow

---

## Need Help?

- Check [DOCKER_COMPOSE_SETUP.md](./DOCKER_COMPOSE_SETUP.md) for detailed setup guide
- Review logs: `docker-compose logs ocg-server`
- See [README.md](./README.md) for platform features
- Check [docs/](./docs/) for operational guides

