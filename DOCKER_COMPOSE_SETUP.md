# OCG Docker Compose Deployment Guide

## Prerequisites

You need:
1. **Docker & Docker Compose** installed
2. **Existing PostgreSQL cluster** (or create one)
   - Must have a database and user ready
   - Network access from Docker containers
3. **Brevo SMTP credentials** (free tier available)
4. **GitHub OAuth app** (if using GitHub login)
5. **A reverse proxy** (nginx, traefik, caddy, etc.) pointing to `http://localhost:9000`

---

## Step 1: Prepare Your PostgreSQL Cluster

Make sure your cluster has a database and user ready for OCG. You can skip this if you already have them set up.

### Create Database and User

Connect to your PostgreSQL cluster as a superuser (e.g., `postgres`):

```bash
# Connect to your cluster
psql -h your-postgres-host.com -U postgres -d postgres

# Inside psql, run:
CREATE USER ocg WITH PASSWORD 'your_secure_password_here';
CREATE DATABASE ocg OWNER ocg;
GRANT ALL PRIVILEGES ON DATABASE ocg TO ocg;

# For newer PostgreSQL (v15+), also run:
GRANT ALL ON SCHEMA public TO ocg;

# Exit psql
\q
```

**Tips:**
- Use a strong, random password (min 32 chars for production)
- If you already have a user/database, just verify the password and permissions
- Test the connection from your Docker host:
  ```bash
  psql -h your-postgres-host.com -U ocg -d ocg -c "SELECT 1"
  ```

---

## Step 2: Get Brevo SMTP Credentials

1. Go to [brevo.com](https://www.brevo.com) and create a free account
2. Navigate to **Settings > SMTP & API**
3. Under SMTP, you'll see:
   - **SMTP Server:** `smtp-relay.brevo.com`
   - **Port:** `587`
   - **Username:** Your email (e.g., `your-email@example.com`)
   - **Password:** Generate or find your SMTP key
4. You can send up to 300 emails/day on the free tier

---

## Step 3: Create GitHub OAuth App (Optional)

Only do this if you want GitHub login enabled. You can always skip this and enable it later.

### Register OAuth App in GitHub

1. Go to your GitHub org/account settings
2. Navigate to **Settings → Developer settings → OAuth Apps**
3. Click **New OAuth App**
4. Fill in the form:
   - **Application name:** `Open Community Groups` (or whatever you prefer)
   - **Homepage URL:** `https://yourdomain.com` (your reverse proxy hostname)
   - **Application description:** (optional) `Community event management platform`
   - **Authorization callback URL:** `https://yourdomain.com/log-in/oauth2/github/callback`
5. Click **Register application**
6. You'll see your **Client ID** — copy it
7. Click **Generate a new client secret** and copy it (you'll only see it once)

### Store Credentials

Save these in a safe place or directly in your `.env`:
```env
GITHUB_OAUTH_ENABLED=true
GITHUB_CLIENT_ID=your_client_id_here
GITHUB_CLIENT_SECRET=your_client_secret_here
```

**Note:** The callback URL **must** match exactly (including `https://` and the trailing path). Update it if your domain changes.

---

## Step 4: Create `.env` File

Create a `.env` file in the repo root with your secrets:

```env
# Database (your external PostgreSQL cluster)
DB_HOST=your-postgres-host.example.com
DB_PORT=5432
DB_NAME=ocg
DB_USER=ocg
DB_PASSWORD=your_secure_db_password

# Server configuration
SERVER_BASE_URL=https://yourdomain.com
COOKIE_SECURE=true

# Email (Brevo)
EMAIL_FROM_ADDRESS=noreply@yourdomain.com
EMAIL_FROM_NAME=Open Community Groups
SMTP_HOST=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_USERNAME=your-brevo-email@example.com
SMTP_PASSWORD=your_brevo_smtp_key

# GitHub OAuth (optional - leave blank if not using)
GITHUB_OAUTH_ENABLED=true
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret

# Image storage (use "db" for simplicity, or "s3" if you have S3)
IMAGE_PROVIDER=db

# Logging
LOG_FORMAT=json
```

⚠️ **Database Connection:**
- `DB_HOST` — hostname of your PostgreSQL cluster (must be reachable from Docker)
- `DB_PORT` — usually 5432
- `DB_NAME`, `DB_USER`, `DB_PASSWORD` — credentials for a database/user you've created

---

## Step 5: Configure Your Reverse Proxy

Example for **nginx**:

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

Example for **Traefik** (docker-compose):
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.ocg.rule=Host(`yourdomain.com`)"
  - "traefik.http.routers.ocg.entrypoints=websecure"
  - "traefik.http.services.ocg.loadbalancer.server.port=9000"
```

---

## Step 6: Start the Stack

```bash
# Build and start all services
docker-compose up -d

# Watch the logs
docker-compose logs -f ocg-server

# Check health
docker-compose ps
```

**First run will:**
1. Run migrations on your external cluster (creates tables, functions, indexes)
2. Start the server on port 9000

**Expected output after ~30s:**
```
ocg-server    | Server running on http://0.0.0.0:9000
```

---

## Step 7: Verify Deployment

1. **Via curl (local testing):**
   ```bash
   curl http://localhost:9000
   ```

2. **Via your domain:**
   ```
   https://yourdomain.com
   ```

3. **Test email:** Try signing up with an email address — you should receive a confirmation email

---

## Common Environment Variables

| Variable | Required | Notes |
|----------|----------|-------|
| `DB_HOST` | ✓ | PostgreSQL hostname (your cluster) |
| `DB_PORT` | ✓ | PostgreSQL port (usually 5432) |
| `DB_NAME` | ✓ | Database name |
| `DB_USER` | ✓ | Database user |
| `DB_PASSWORD` | ✓ | Database password |
| `OCG_SERVER__BASE_URL` | ✓ | Your public domain |
| `OCG_EMAIL__SMTP__*` | ✓ | SMTP credentials |
| `OCG_SERVER__LOGIN__EMAIL` | — | Enable email/password login (default: true) |
| `OCG_SERVER__LOGIN__GITHUB` | — | Enable GitHub OAuth (default: false) |
| `OCG_IMAGES__PROVIDER` | — | `db` or `s3` (default: db) |
| `COOKIE_SECURE` | — | HTTPS only (default: true) |

---

## Useful Commands

```bash
# View logs
docker-compose logs -f ocg-server

# Stop everything
docker-compose down

# Restart just the server
docker-compose restart ocg-server

# Rebuild images
docker-compose build --no-cache

# Check service health
docker-compose ps
```

---

## Troubleshooting

### Server won't start
```bash
docker-compose logs ocg-server
# Check if BASE_URL, SMTP, and DB credentials are correct
```

### Can't send emails
- Verify Brevo SMTP credentials in `.env`
- Check if you're hitting the 300 email/day free tier limit
- Look for errors in logs: `docker-compose logs ocg-server | grep -i email`

### Database migrations failed
```bash
docker-compose logs db-migrator
# Re-run: docker-compose up db-migrator
```

### Can't connect to PostgreSQL cluster
- Verify `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` in `.env`
- Ensure your cluster is reachable from your Docker host
- Check if the database/user exists: `psql -h DB_HOST -U DB_USER -d DB_NAME`

---

## Next Steps

- Set up GitHub OAuth (optional, follow Step 2)
- Configure your domain DNS to point to your reverse proxy
- Create your first community/group via the web UI
- Add admins via the database or invite system

