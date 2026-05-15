# Quick Start Guide - Development vs Production

## Development (Local)

Everything included locally with zero setup needed:

```bash
# Clone repo
git clone <repo>
cd open-community-groups

# One-liner startup
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f ocg-server
```

**What's included:**
- PostgreSQL 18 (user: `ocg`, pass: `ocg_dev`, db: `ocg`)
- Mailhog for SMTP dev (Web UI: http://localhost:8025)
- OCG Server with email/password login enabled
- Pretty-printed logs

**Access:**
- **App:** http://localhost:9000
- **Email UI:** http://localhost:8025 (to see test emails)
- **Database:** `psql -h localhost -U ocg -d ocg` (password: `ocg_dev`)

**Sign up:**
1. Go to http://localhost:9000
2. Click "Sign up" or "Log in"
3. Use email/password
4. Check http://localhost:8025 for verification email

**Cleanup:**
```bash
docker-compose down          # Stop containers
docker-compose down -v       # Stop + remove volumes (clean slate)
```

**Customize (optional):**
- Edit `.env.development` if you want to override defaults
- Or just use the defaults in `docker-compose.yml`

---

## Production (External DB)

Use your own PostgreSQL cluster + Brevo email:

```bash
# Prepare PostgreSQL
psql -h your-postgres.com -U postgres -d postgres
# CREATE USER ocg WITH PASSWORD 'strong_password';
# CREATE DATABASE ocg OWNER ocg;
# GRANT ALL PRIVILEGES ON DATABASE ocg TO ocg;

# Copy env template
cp .env.example .env

# Fill in your production values
# - DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
# - SERVER_BASE_URL
# - SMTP_HOST, SMTP_USERNAME, SMTP_PASSWORD (Brevo)
# - GITHUB_CLIENT_ID, GITHUB_CLIENT_SECRET (optional)

# Start with prod compose
docker-compose -f docker-compose.prod.yml up -d

# View logs
docker-compose -f docker-compose.prod.yml logs -f ocg-server
```

**Key differences:**
- External PostgreSQL cluster (managed separately)
- External SMTP (Brevo, SendGrid, etc.)
- HTTPS/SSL termination at reverse proxy
- Optional GitHub OAuth
- JSON structured logs

**Reverse proxy setup:**
Point your nginx/Traefik/Caddy to `http://localhost:9000`

---

## Compose Files

| File | Purpose | DB | Email | Setup Time |
|------|---------|----|----|---|
| `docker-compose.yml` | Local development | Local PG18 | Mailhog | ~1 minute |
| `docker-compose.prod.yml` | Production | External | External | ~10 minutes |

---

## Environment Files

| File | Purpose | Size |
|------|---------|------|
| `.env.example` | Production template | Full config |
| `.env.development` | Dev overrides | Minimal (optional) |
| `.env` | Your production secrets | Don't commit! |

