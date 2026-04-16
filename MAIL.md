# Mail Configuration

## Architecture

Ghost sends two distinct types of email through two separate paths:

```
Transactional (logins, invites, password resets):
  Ghost → SES SMTP (email-smtp.eu-west-1.amazonaws.com:465)

Newsletter bulk sending:
  Ghost → [thinks it's Mailgun] → ses-adapter:3001 → AWS SES SDK
```

### Key facts
- `ses-adapter` reads AWS credentials from **environment variables only** — `config.json` is mounted but **ignored** by the adapter code
- Credentials and SMTP config live in `.env` (gitignored) — see below
- `mailgun_base_url` in the Ghost DB must point to `http://ses-adapter:3001/v3` — apply with `ses-adapter/plumb-to-db` if it ever resets
- IAM credentials (`AKIA...`) are used directly for the SES SDK (ses-adapter); the SMTP password is **derived** from the IAM secret — it is NOT the raw secret key

### SMTP password derivation

The SES SMTP password is derived from the IAM secret access key via AWS Signature v4. To regenerate it:

```python
import hmac, hashlib, base64, json

def derive_smtp_password(secret_key, region):
    date, service, terminal, message = "11111111", "ses", "aws4_request", "SendRawEmail"
    kDate     = hmac.new(("AWS4" + secret_key).encode(), date.encode(),     hashlib.sha256).digest()
    kRegion   = hmac.new(kDate,    region.encode(),                         hashlib.sha256).digest()
    kService  = hmac.new(kRegion,  service.encode(),                        hashlib.sha256).digest()
    kTerminal = hmac.new(kService, terminal.encode(),                       hashlib.sha256).digest()
    kMessage  = hmac.new(kTerminal, message.encode(),                       hashlib.sha256).digest()
    return base64.b64encode(b'\x04' + kMessage).decode()

cfg = json.load(open("ses-adapter/config.json"))
print(derive_smtp_password(cfg["aws"]["secretAccessKey"], cfg["aws"]["region"]))
```

## Required `.env` entries

```bash
# Transactional mail (logins, password resets, invites)
mail__transport=SMTP
mail__options__host=email-smtp.eu-west-1.amazonaws.com
mail__options__port=465
mail__options__service=SES
mail__options__secure=true
mail__options__auth__user=YOUR_IAM_ACCESS_KEY_ID
mail__options__auth__pass=YOUR_DERIVED_SMTP_PASSWORD   # see derivation above
mail__from="'Your Name' <you@yourdomain.com>"

# Newsletter bulk sending (passed to ses-adapter container)
SES_ACCESS_KEY_ID=YOUR_IAM_ACCESS_KEY_ID
SES_SECRET_ACCESS_KEY=YOUR_IAM_SECRET_ACCESS_KEY
```

## Diagnostics

### Check all containers are up

```bash
docker compose ps
```

Expected: `ghost`, `db`, `caddy`, `ses-adapter` all `Up`.

### Verify ses-adapter has credentials and correct region

```bash
docker compose logs ses-adapter | head -10
```

Expected startup output:
```
Ghost-to-SES adapter running at http://0.0.0.0:3001
AWS Region: eu-west-1
Default sender: hello@yourdomain.com
```

If region shows `us-east-1`, the env vars aren't being passed — check `compose.yml` and `.env`.

### Verify DB setting

```bash
sudo bash -c 'source .env && docker exec ghost-docker-db-1 mysql -u ghost -p"${DATABASE_PASSWORD}" ghost \
  -e "SELECT \`key\`, value FROM settings WHERE \`key\` = '"'"'mailgun_base_url'"'"';"'
```

Expected: `http://ses-adapter:3001/v3`. If wrong or missing:

```bash
bash ses-adapter/plumb-to-db
```

### Test Ghost → ses-adapter connectivity

```bash
docker compose exec ghost wget -qO- http://ses-adapter:3001/
```

Expected: `{"message":"Success"}`

### Check Ghost logs for mail errors

```bash
docker compose logs ghost | grep -i "mail\|error"
```

### Trigger a test email

Send a staff invite from Ghost Admin, then:

```bash
docker compose logs ses-adapter --tail=20
docker compose logs ghost --tail=20
```

### All relevant logs at once

```bash
docker compose logs --tail=50 ghost ses-adapter db caddy
```
