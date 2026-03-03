# SES Mail Diagnostics

## Architecture (current approach)

```
Ghost → [thinks it's Mailgun] → ses-adapter:3001 → AWS SES
```

- `ses-adapter` container builds from `ses-adapter/Dockerfile` (clones `exlab-code/ghost-cms-amazon-ses-adapter` at build time)
- `ses-adapter/config.json` is mounted read-only — **this file must exist on the server, it is gitignored**
- `mysql-init/02-ghost-ses-adapter.sql` sets Ghost's `mailgun_base_url` to `http://ses-adapter:3001/v3`
- `ses-adapter/plumb-to-db` is a helper script to apply the SQL against the running DB

## Step 1 — Check containers are up

```bash
docker compose ps
```

Expected: `ghost`, `db`, `caddy`, `ses-adapter` all `Up`. If `ses-adapter` is `Exit` or missing, that's your first problem.

## Step 2 — Check ses-adapter logs

```bash
docker compose logs ses-adapter
```

Look for:
- Build errors (GitHub clone fails if no internet at build time)
- `config.json` missing or malformed (mount failure = container won't start)
- Startup errors from `ghost-ses-adapter.js`

## Step 3 — Verify config.json exists on the server

```bash
ls -la ses-adapter/config.json
cat ses-adapter/config.json
```

It must exist (gitignored, so not pulled from VCS). Template is at `ses-adapter/config.json-example`:

```json
{
  "port": 3001,
  "aws": {
    "accessKeyId": "YOUR_ACCESS_KEY_ID",
    "secretAccessKey": "YOUR_SECRET_ACCESS_KEY",
    "region": "eu-west-1"
  },
  "defaultSender": "you@yourdomain.com",
  "logLevel": "normal"
}
```

## Step 4 — Verify the DB setting is applied

```bash
source .env
docker exec ghost-docker-db-1 mysql -u ghost -p"${DATABASE_PASSWORD}" ghost \
  -e "SELECT \`key\`, value FROM settings WHERE \`key\` = 'mailgun_base_url';"
```

Expected output:
```
+-----------------+---------------------------+
| key             | value                     |
+-----------------+---------------------------+
| mailgun_base_url | http://ses-adapter:3001/v3 |
+-----------------+---------------------------+
```

If missing or wrong, run the plumb script:
```bash
cd ses-adapter && bash plumb-to-db
```

## Step 5 — Check Ghost mail config in .env

Ghost needs these set (the adapter handles the actual sending; Ghost just needs to know it's using "Mailgun"):

```bash
grep -i mail .env
```

You should have something like:
```
mail__transport=Mailgun
mail__options__apiKey=any-non-empty-string
mail__from="'Your Name' <noreply@yourdomain.com>"
```

> Note: The `apiKey` value doesn't matter — the adapter ignores it — but Ghost requires it to be non-empty to attempt sending.

## Step 6 — Check Ghost logs for mail errors

```bash
docker compose logs ghost | grep -i mail
docker compose logs ghost | grep -i error
```

## Step 7 — Test connectivity between Ghost and ses-adapter

```bash
docker compose exec ghost wget -qO- http://ses-adapter:3001/
```

If that fails, the containers can't reach each other (network issue).

## Step 8 — Trigger a test email

In Ghost Admin → Settings → Email, send a test. Then immediately:
```bash
docker compose logs ses-adapter --tail=50
docker compose logs ghost --tail=50
```

---

## Alternative: Native SES via SMTP (simpler, no sidecar)

Skip the adapter entirely. AWS SES provides an SMTP endpoint. In `.env`:

```bash
mail__transport=SMTP
mail__options__host=email-smtp.eu-west-1.amazonaws.com   # adjust region
mail__options__port=587
mail__options__secure=false
mail__options__auth__user=YOUR_SES_SMTP_USERNAME
mail__options__auth__pass=YOUR_SES_SMTP_PASSWORD
mail__from="'Your Name' <noreply@yourdomain.com>"
```

SES SMTP credentials are generated separately from IAM keys: AWS Console → SES → SMTP Settings → Create SMTP Credentials.

This removes the `ses-adapter` container and the DB patching entirely. Worth considering if the adapter continues to cause grief.

---

## Quick reference — all relevant logs at once

```bash
docker compose logs --tail=50 ghost ses-adapter db caddy
```
