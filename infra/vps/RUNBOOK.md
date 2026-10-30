# Operations runbook: single VPS (ADR-0012)

Commands marked **[Mac]** run on your computer, **[server]** over SSH as user `occpl`
(`ssh -i ~/.ssh/occpl_vps occpl@<server-ip>`), unless stated otherwise.

## 1. Before you start

| You need | Notes |
|---|---|
| A VPS | Ubuntu 24.04, **4 vCPU, 8 GB RAM or more, 80 GB+ disk**, IPv4 address, in an Indian data centre if offered (latency, data residency). Add your SSH public key when ordering. |
| The domain's DNS in Route 53 | Done (zone `occplsoftech.com`). |
| A GitHub admin login | For the environment, variable and secrets in §3. |
| An SMTP mailbox | `support@occplsoftech.com` (Hostinger mail). |

Generate a key pair for the server: **[Mac]** `ssh-keygen -t ed25519 -f ~/.ssh/occpl_vps -C occpl-deploy`
(use it for the VPS and as the deploy key in §3; `~/.ssh/occpl_vps.pub` is the public half).

## 2. DNS: point the three hosts at the server

Route 53 → hosted zone `occplsoftech.com` → **Create record**, three **A** records, TTL 300, value =
the server's IPv4 address:

| Record name | Type | Value |
|---|---|---|
| `admin` | A | `<server-ip>` |
| `api` | A | `<server-ip>` |
| `auth` | A | `<server-ip>` |

Certificates are requested on first start, so these must resolve to the server (check with
`https://dnschecker.org`, type A) before the first deploy.

## 3. GitHub

Repository → Settings → Environments → **production** (create it; deployment branch `main`; add
yourself as **required reviewer**). In it:

| Kind | Name | Value |
|---|---|---|
| Variable | `VPS_HOST` | the server's IPv4 address |
| Secret | `VPS_SSH_KEY` | the content of `~/.ssh/occpl_vps` (private key) |
| Secret | `VPS_KNOWN_HOSTS` | output of `ssh-keyscan -t ed25519 <server-ip>` |

Also check Settings → Actions → General → *Workflow permissions* allows the workflow to publish
packages (the workflow declares `packages: write`).

## 4. Prepare the server (once)

1. **[Mac]** Copy the bundle: from the repository root,
   ```bash
   ssh -i ~/.ssh/occpl_vps root@<server-ip> 'mkdir -p /opt/occpl/backup'
   scp -i ~/.ssh/occpl_vps -r infra/vps/. root@<server-ip>:/opt/occpl/
   scp -i ~/.ssh/occpl_vps -r infra/backup/. root@<server-ip>:/opt/occpl/backup/
   ```
   (Hostinger's VPS login is `root` with your key. Use the panel's console if SSH is not up yet.)
2. **[server as root]** `bash /opt/occpl/bin/install.sh`. It installs Docker, opens only ports
   22/80/443, enables fail2ban and automatic security updates, creates the 2 GB swap, creates the
   `occpl` user (same SSH key) and the cron jobs.
3. **[server as occpl]** `cd /opt/occpl && bin/init-env.sh`. It asks for the domain, certificate
   email, **the public IP addresses allowed to open the Keycloak console** (your office/home IP,
   `curl -s https://checkip.amazonaws.com` on your computer, with `/32`), the SMTP mailbox
   and password, and the first Super Admin's email. It writes `/opt/occpl/.env` (mode 600).
   **Copy `.env` to a password manager**: it holds every password.
4. SMTP mode: Hostinger usually accepts port 465 (SSL, the default here); if the mailbox only
   works on 587, answer `587` (STARTTLS).

## 5. First deploy and first Super Admin

1. Merge to `main` (or Actions → *Release* → Run workflow is only for an existing build). CI runs,
   then *Release* publishes the images and waits for your approval: **Review deployments →
   production → Approve**. It copies the bundle, runs `bin/deploy.sh` (first start pulls images and
   migrates the database: about 2 minutes) and the smoke test.
2. Open `https://auth.occplsoftech.com/admin/` **from an allowed IP**. User `bootstrap-admin`,
   password = `KEYCLOAK_ADMIN_PASSWORD` in `/opt/occpl/.env`. In realm **occpl**: Users → Add user, the
   email you gave as the first Super Admin (mark *Email verified*), Credentials → set a password (or
   *Required user actions → Update password* and send the email).
3. The user opens `https://admin.occplsoftech.com`, signs in and sets up an authenticator app; Core
   then grants SUPER_ADMIN (audited).
4. **[server]** Clear `BOOTSTRAP_SUPER_ADMIN_EMAIL=` in `.env`, then `docker compose up -d core-platform`.
5. Create a personal admin in the Keycloak *master* realm and disable `bootstrap-admin`.

## 6. Backups, recovery, monitoring

**Off-host backups.** In the AWS console/CLI on your Mac: `cd infra/aws/shared && terraform apply`
creates the bucket and a write-only user. Then `terraform output backups_bucket`,
`terraform output vps_backup_access_key_id` and `terraform output -raw vps_backup_secret_access_key`.
**[server as occpl]** `aws configure` (region `ap-south-1`, the two values), then add
`BACKUP_S3_URI=s3://<bucket>/vps` to `.env` and run `bin/backup.sh full`; check with
`aws s3 ls s3://<bucket>/vps/`.

Cron (installed by `install.sh`): full backup 02:00 IST, binary-log increment every 5 minutes, health
check every 5 minutes; 7 days are kept locally, 35 in S3.

**Alerts.** Create a free check at healthchecks.io (period 10 min), put its ping URL in `.env`
as `HEARTBEAT_URL=...`. It emails you when the server stops reporting or a service is unhealthy. Add
an external uptime monitor (e.g. UptimeRobot) on `https://admin.occplsoftech.com/healthz`. Set
`BACKUP_PING_URL` to a second healthchecks.io check (period 1 day) to be told about a missed
nightly backup.

**Restore (new or damaged server; tested, 6–8 s for the drill database).**
1. Prepare a server (§4 steps 1–3, restore the saved `.env` instead of running `init-env.sh`).
2. **[server]** Fetch the backups: `aws s3 sync s3://<bucket>/vps /var/backups/occpl`.
3. Start only the database: `cd /opt/occpl && docker compose up -d mysql` (wait until healthy).
4. Restore (latest point, or add `STOP_BEFORE=<gtid>` to stop before a mistake):
   ```bash
   MYSQL_HOST=127.0.0.1 MYSQL_USER=root MYSQL_PASSWORD=$(sed -n 's/^MYSQL_ROOT_PASSWORD=//p' .env) \
     backup/restore.sh /var/backups/occpl
   ```
5. Start the rest: `bin/deploy.sh <the release that was running>` (see `RELEASE=` in `.env`), then take a
   full backup: `bin/backup.sh full`.

## 7. Day-to-day

- **Deploy:** merge to `main`, approve in GitHub. **Roll back:** Actions → *Release* → Run workflow with
  the full SHA of an earlier commit (the images are kept in the registry), approve. A failed deploy
  rolls back by itself.
- **Logs:** `docker compose logs -f core-platform` (also `keycloak`, `caddy`, `mysql`); deploy log
  `/var/log/occpl-deploy.log`; health problems `/var/log/occpl-health.log`.
- **Status:** `docker compose ps`; disk: `df -h /` (backups and images grow it).
- **OS updates:** security updates install automatically; every few weeks run
  `sudo apt-get update && sudo apt-get upgrade -y && sudo reboot` (about a minute of downtime; the
  containers restart by themselves).
- **Secrets:** edit `/opt/occpl/.env`, then `docker compose up -d`. Database passwords also have to be
  changed inside MySQL (`ALTER USER ...`); Keycloak client secrets in the realm (Clients → Credentials).
- **Razorpay:** put the keys into `.env`, `docker compose up -d core-platform`; set the webhook URL
  to `https://api.occplsoftech.com/api/v1/webhooks/payments/razorpay`.
- **Realm changes after the first start:** the realm import only runs when the realm does not exist:
  change settings in the Keycloak console.
- **Resize:** upgrade the VPS plan in the provider's panel (a reboot), then raise
  `CORE_MEM_LIMIT`, `KEYCLOAK_MEM_LIMIT`, `MYSQL_MEM_LIMIT` and `MYSQL_BUFFER_POOL` in `.env`.

## 7a. Customer shipping portal (courier profile)

A separate web app for a courier company's business customers (book, compare rates, track, labels). It runs next to the
courier containers and talks to the same backend; repository `occplsoftech/e-commerce-bussines`.

1. **DNS:** an A/AAAA record for `PORTAL_HOST` (default `portal.<courier host without its first label>`, set by
   `bin/courier-setup.sh`; this installation uses `e-commerce.occplsoftech.com` -> 65.0.187.198, so set `PORTAL_HOST=e-commerce.occplsoftech.com` in `.env`) pointing at this server, like the other hosts. Caddy gets the certificate by itself.
2. **`.env`:** `PORTAL_HOST` (written by the setup script), optionally `PORTAL_COMPANY_CODE` (the company whose customers
   sign in; blank shows a "Company code" field), `PORTAL_COMPANY_NAME`, `PORTAL_SUPPORT_EMAIL`. Then
   `docker compose up -d` starts `courier-portal` and restarts the backend with the portal's origin in its CORS list and
   `CUSTOMER_PORTAL_BASE_URL` (the links in activation and password-reset emails).
   **After changing `PORTAL_HOST` (or `COURIER_HOST`) in `.env`, recreate Caddy:** `docker compose up -d caddy`. Caddy reads these
   values only when its container is created, so until then it serves the old host and the portal's HTTPS fails with
   `tlsv1 alert internal error` (the release smoke test shows exactly that). Check with
   `docker compose exec caddy printenv PORTAL_HOST` and `docker compose logs caddy | grep -i obtain`.
3. **Couriers (optional):** a company administrator connects the company's own courier accounts in the courier app, under
   *Administration > Courier Integrations* (Shiprocket: API user e-mail, password, pickup location; Delhivery: API token, pickup
   location), then presses *Test connection*. Secrets are stored encrypted and never shown again. Blue Dart and Ekart are listed as
   "Coming soon" (no integration code yet). Use a Shiprocket **API user**, not the owner login. *Fallback:* settings in `.env`
   (`SHIPROCKET_ENABLED`, `SHIPROCKET_API_EMAIL`, `SHIPROCKET_API_PASSWORD`, `SHIPROCKET_PICKUP_LOCATION`, `DELHIVERY_ENABLED`,
   `DELHIVERY_API_TOKEN`, `DELHIVERY_PICKUP_LOCATION`) apply to every company that has not saved its own account.
   The markup is the administrator-managed rule in the courier app (`/api/v1/pricing-markup-rules`,
   default 10 % on external couriers); GST on that markup and on OCCPL's network is `app.customer-portal.markup-gst-percent`
   / `occpl-network-gst-percent` (18 by default - confirm with the accountant).
   **Status updates:** the backend checks every booked shipment with its courier hourly by itself (`TRACKING_SYNC_ENABLED`,
   on by default). For instant updates, copy the *Webhook address* shown on each courier's card in Courier Integrations
   into the courier's panel (Shiprocket: Settings > API > Webhooks; Delhivery: ask the account manager to enable status
   push to that URL). The address is built from `APP_BASE_URL` (`https://$COURIER_HOST`, set in the compose file).
4. **Deploys:** the portal repository's Release workflow runs `courier-deploy.sh portal <sha>` over the same restricted SSH
   key as the courier repositories (environment `dev`: variables `VPS_HOST`, `PORTAL_URL`; secrets `VPS_SSH_KEY`,
   `VPS_KNOWN_HOSTS`). Rollback: run that workflow by hand with an earlier full commit SHA.
5. **Customer accounts:** a company administrator grants portal access to a business customer in the courier app
   (`POST /api/v1/customers/{id}/portal-access`); the customer activates it from the emailed link.


## 7b. Courier e-mail (SMTP) and KYC codes

The courier backend writes account e-mails (activation, password reset) and KYC codes to its log until real
e-mail is switched on. To send them for real, add to `/opt/occpl/.env` and recreate the backend:

```
COURIER_EMAIL_ENABLED=true
COURIER_SMTP_HOST=smtp.gmail.com        # or your provider's SMTP host
COURIER_SMTP_PORT=587
COURIER_SMTP_USERNAME=you@yourdomain.com
COURIER_SMTP_PASSWORD=<app password>    # never commit it
COURIER_EMAIL_FROM=you@yourdomain.com   # optional; defaults to the SMTP user name
```

```
docker compose --profile courier up -d courier-backend
```

Mobile (SMS) codes go through each company's own SMS channel (courier-web → Communication Center → Channel
Settings). Until a company sets one up, mobile KYC codes are refused with a clear message. On the dev server only,
`COURIER_KYC_LOG_CODES=true` also writes every KYC code to the backend log
(`docker compose logs courier-backend | grep "DEV KYC CODE"`).

## 8. Moving to AWS later

`infra/aws` is ready (ADR-0011). Restore the latest backup into RDS with `backup/restore.sh`
(RUNBOOK of `infra/aws` §3), point DNS at the load balancer, switch the pipeline to
`deploy-aws.yml`.

## 9. Development/test server on EC2 (D-17)

The same stack runs on one EC2 instance at `admin.dev.occplsoftech.com`, `api.dev…`, `auth.dev…`
(about $85 a month while running; **stop the instance in the console when not in use**: compute stops
billing, the disk and the public address still do). It is the place to try a release before approving
the production deploy. Outgoing email is real (the same SMTP mailbox): use test addresses.

1. **[Mac]** `cd infra/aws/dev && cp terraform.tfvars.example terraform.tfvars`; fill in your public
   key (`cat ~/.ssh/occpl_vps.pub`) and your public IP (`curl -s https://checkip.amazonaws.com`,
   as `x.x.x.x/32`). `terraform init && terraform apply` (needs the bootstrap stack; creates the
   instance, a fixed public address, firewall rules, an S3-backup role and the three DNS records).
   Optional but recommended: apply `infra/aws/shared` first (backup bucket).
2. Copy the bundle (the login user on Ubuntu is `ubuntu`):
   ```bash
   scp -i ~/.ssh/occpl_vps -r infra/vps ubuntu@<ip>:/tmp/occpl
   scp -i ~/.ssh/occpl_vps -r infra/backup ubuntu@<ip>:/tmp/occpl/backup
   ssh -i ~/.ssh/occpl_vps ubuntu@<ip> 'sudo mkdir -p /opt/occpl && sudo cp -r /tmp/occpl/. /opt/occpl/ && sudo bash /opt/occpl/bin/install.sh'
   ```
3. `ssh -i ~/.ssh/occpl_vps occpl@<ip>`, then `cd /opt/occpl && bin/init-env.sh` with domain
   `dev.occplsoftech.com` (the `domain_for_env` output) and your IP as ADMIN_IPS. For backups add
   `BACKUP_S3_URI=<backup_s3_uri output>` to `.env` (the instance role grants access; no `aws configure`).
4. GitHub environment **dev** (no reviewers needed): variables `VPS_HOST` = the public address and
   `HOST_SUFFIX` = `dev.`; secrets `VPS_SSH_KEY` and `VPS_KNOWN_HOSTS` as in §3. After a merge to `main`
   the *Release* workflow deploys to dev automatically and smoke-tests it; production only follows when
   the repository variable `DEPLOY_PRODUCTION` is `true` and the reviewers approve.
5. The first Super Admin and the rest work as in §5, with the `.dev.` hostnames.
6. The GitHub runners connect from changing addresses: the security group must allow SSH (22) from anywhere
   (key-only login, fail2ban), plus 80 and 443. If you create the instance by hand in the console instead of
   Terraform: Ubuntu 24.04, `t3.large`, 80 GiB gp3, Elastic IP, credit specification *Standard*, IMDSv2 only; log in
   as `ubuntu` with the key pair you created, add `~/.ssh/occpl_vps.pub` to `~/.ssh/authorized_keys`, then continue at step 2.
7. **Sign-in page theme on an existing realm.** The realm import only runs on the very first start, so a realm created
   before the `occpl` theme existed keeps the stock Keycloak look. Set it once on the server (after the release that
   contains the theme is deployed):
   ```bash
   cd /opt/occpl && set -a && . ./.env && set +a
   K="docker exec -i $(docker ps --format '{{.Names}}' | grep keycloak | head -1) /opt/keycloak/bin/kcadm.sh"
   $K config credentials --server http://localhost:8080 --realm master --user bootstrap-admin --password "$KEYCLOAK_ADMIN_PASSWORD"
   $K update realms/occpl -s loginTheme=occpl
   ```
   New installs get it from the generated production realm.
8. **One-time code off on dev only (optional).** Add `MFA_PLATFORM_REQUIRED=false` to `/opt/occpl/.env`, then
   `docker compose up -d core-platform`. Core then gives platform staff their permissions without the authenticator
   code. To stop Keycloak asking for the code, delete the user's *otp* credential (Keycloak admin console, realm
   occpl, Users, Credentials). Do not do this on production: MFA for platform roles is a security requirement (D-2).
9. **Admin console open to every address on dev (optional, dev only).** Set `ADMIN_IPS=0.0.0.0/0 ::/0` in `/opt/occpl/.env` and
   `docker compose up -d --force-recreate caddy`; then add the variable `ALLOW_PUBLIC_ADMIN_CONSOLE` = `true` to the GitHub
   environment **dev** (Settings, Environments, dev), otherwise the smoke test reports the open console as a failure. Anyone can then reach
   the Keycloak administrator sign-in page, so keep `KEYCLOAK_ADMIN_PASSWORD` long and random. Never on production.
10. **Register the courier product (dev).** Two parts; do the Keycloak part first. Use your own hostnames if they differ.
    *Keycloak* (on the server, after the `kcadm config credentials` line from step 7). This creates a client scope named
    like the product's `apiAudience`, gives it an audience mapper, and lets `occpl-core-services` request it per call:
    ```bash
    SCOPE=$($K create client-scopes -r occpl -i -s name=courier-api -s protocol=openid-connect \
      -s 'attributes."include.in.token.scope"=false')
    $K create client-scopes/$SCOPE/protocol-mappers/models -r occpl -s name=courier-api-audience \
      -s protocol=openid-connect -s protocolMapper=oidc-audience-mapper \
      -s 'config."included.custom.audience"=courier-api' -s 'config."access.token.claim"=true' \
      -s 'config."id.token.claim"=false'
    CID=$($K get clients -r occpl -q clientId=occpl-core-services --fields id --format csv --noquotes)
    $K update clients/$CID/optional-client-scopes/$SCOPE -r occpl
    ```
    *Portal* (Super Admin, Products, New product): code `courier`, name `Courier Management`, app URL
    `https://courier.dev.occplsoftech.com`, API base URL `https://courier.dev.occplsoftech.com` (the same host: Caddy sends `/provisioning/*` to the backend), API audience
    `courier-api`. Then add a configuration version
    `{"schemaVersion":1,"launch":{"path":"/"},"provisioning":{"mode":"SYNC","path":"/provisioning/v1/tenants/{companyId}"}}`,
    publish it and activate the product.
11. **Host the courier app (dev).** It runs beside the platform on this server, using this server's MySQL (own database
    `courier_db` and user) and Redis, and is off unless `COMPOSE_PROFILES` contains `courier`.
    1. DNS: add an A record `courier.dev.occplsoftech.com` pointing at the server (same address as the other hosts).
    2. Images: the workflow *Image* in each of the two courier repositories publishes `ghcr.io/occplsoftech/courier-service`
       (the API, repository `courier-service`) and `ghcr.io/occplsoftech/courier-web` (the web app, repository `courier-web`), tags: the commit SHA and `dev`. Every deploy logs the server into `ghcr.io` with the workflow's token, which
       cannot read private packages of other repositories. Add the repository secret `GHCR_PULL_TOKEN` (GitHub, Settings,
       Secrets and variables, Actions): a *classic* personal access token of the account that owns the packages, with only
       `read:packages`. The deploy then uses it; without it the courier images fail to pull with 403. If the token's account is
       not the repository owner, also add the repository variable `GHCR_PULL_USER` with that account's username.
    3. Settings and database, once. The plan code is the courier plan you create in step 3 below (it must exist before
       the first company is provisioned):
       ```bash
       cd /opt/occpl && bin/courier-setup.sh courier.dev.occplsoftech.com STARTER
       docker compose pull && docker compose up -d
       docker compose logs --tail 80 courier-backend      # Flyway runs the migrations on first start
       ```
       The script keeps existing values in `.env`, generates the courier's database password, JWT secret and encryption key,
       and adds `courier` to `COMPOSE_PROFILES`. From then on every `bin/deploy.sh` also pulls and starts the courier
       containers (tag `dev`), so push courier images before relying on a deploy.
    4. Check: `https://courier.dev.occplsoftech.com` shows the courier sign-in page, and
       `curl -i https://courier.dev.occplsoftech.com/provisioning/v1/tenants/x` answers 401 (the endpoint exists and wants a token).
    5. Memory: with Core, Keycloak, MySQL, Redis, the portal and the courier backend on one `t3.large`, watch
       `docker stats`; lower `COURIER_MEM_LIMIT` or move up an instance size if the server swaps.
    6. The courier app starts with no users and no plans. Create the first Super Admin without SQL or sharing a password: add
       `COURIER_BOOTSTRAP_SUPER_ADMIN_EMAIL=you@company.com` and `COURIER_BOOTSTRAP_SUPER_ADMIN_PASSWORD=<a strong password>` to `/opt/occpl/.env`
       (the `courier-service` image must include the `SuperAdminBootstrap` change), run `docker compose up -d courier-backend`, check
       `docker compose logs courier-backend | grep -i bootstrap`, then **delete both lines** from `.env` and run `docker compose up -d courier-backend` again.
       Sign in at the courier host with that account and create the subscription plan whose code you gave to `bin/courier-setup.sh`
       (details: `MEMORY/modules/platform-integration.md` in the `courier-service` repository.
12. **Company logos and favicons in S3 (dev).** Optional; without it the company pages work and only image upload answers
    "File storage is not configured".
    1. In the AWS console create a **private** bucket in the same region as the server (for example `occpl-dev-files`,
       region `ap-south-1`) with *Block all public access* on. No CORS or public policy is needed: the portal shows images
       through short-lived signed links.
    2. Create an IAM user (or role, if the instance has one) with only this policy, replacing the bucket name:
       ```json
       {"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:PutObject","s3:GetObject","s3:DeleteObject"],
         "Resource":"arn:aws:s3:::occpl-dev-files/companies/*"}]}
       ```
    3. On the server add to `/opt/occpl/.env`: `FILES_S3_BUCKET=occpl-dev-files`, `FILES_S3_REGION=ap-south-1` and, for an IAM user,
       `FILES_AWS_ACCESS_KEY_ID=...` and `FILES_AWS_SECRET_ACCESS_KEY=...` (leave both out when the instance role has the policy).
       Then `docker compose up -d core-platform`.
    4. Check: open a company, Profile, upload a logo; it should appear. A signed link stops working after an hour, so reload the page to get a new one.
12b. **File uploads in the courier app (dev).** Without this, the courier answers "File upload is not available: no storage backend is
    configured" for wallet top-up proofs, proof-of-delivery photos and e-way bills.
    1. A private bucket (the same `occpl-dev-files` works) and an IAM user with this policy (the portal's own policy covers only `companies/*`):
       ```json
       {"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:PutObject","s3:GetObject"],
         "Resource":"arn:aws:s3:::occpl-dev-files/*"}]}
       ```
    2. In `/opt/occpl/.env`: `COURIER_FILES_S3_ENABLED=true`, `COURIER_FILES_S3_BUCKET=occpl-dev-files`, `COURIER_FILES_S3_REGION=ap-south-1`,
       `COURIER_FILES_AWS_ACCESS_KEY_ID=...`, `COURIER_FILES_AWS_SECRET_ACCESS_KEY=...` (leave both keys out when the instance role has the policy).
    3. `cd /opt/occpl && docker compose up -d courier-backend`, then `docker compose logs courier-backend | grep -i "S3 file storage"`.
    4. Check: in the e-commerce wallet raise a manual recharge with a proof image; it should upload.

13. **Sign in with OCCPL for the courier app (dev).** People open Courier from the portal and arrive signed in. Needs steps 10 and 11 first.
    1. *Keycloak*, two clients (admin console, realm `occpl`, or `kcadm`):
       - `courier-web`: **public** client (no secret), *Standard flow* on, *PKCE method* `S256`, valid redirect URI
         `https://courier.dev.occplsoftech.com/sso`, web origin `https://courier.dev.occplsoftech.com`; give it the client scope `courier-api`
         (step 10) as a **default** scope so its tokens carry that audience, and make sure the `email` and `profile` scopes are default.
       - `courier-service`: **confidential** client with *Service accounts* on and no other flows. Core uses its client id to know which
         product is calling: put the same value in the courier product's *Service client id* in the portal (Products, courier, edit).
         Add the audience mapper `occpl-core` to it, as for other product service clients (`occpl-realm.json` shows the sample product's).
    2. *Courier product configuration* in the portal: set the launch path to `/sso/{companyId}` (the schema allows no query string; new configuration version, publish).
    3. *Server settings* in `/opt/occpl/.env` (the compose file passes them to `courier-backend`):
       `OCCPL_PLATFORM_CORE_BASE_URL=https://api.dev.occplsoftech.com`,
       `OCCPL_PLATFORM_CORE_TOKEN_URI=https://auth.dev.occplsoftech.com/realms/occpl/protocol/openid-connect/token`,
       `OCCPL_PLATFORM_SERVICE_CLIENT_SECRET=<the courier-service secret>`; then `docker compose up -d courier-backend`.
    4. Check: in the portal open a company that has the courier product and press Open. You should land on the courier dashboard without a
       password. The first time creates a courier user for you (a company admin on the platform becomes a courier company admin; others start as viewers).


14. **Release flow for the courier repositories (dev).** Every merge to `main` of `courier-service` and `courier-web` tests, builds the
    image, deploys it here, runs a smoke test and rolls back if the new container does not become healthy (workflow `Release`;
    run it by hand with a full commit SHA in *release* to roll back). The workflows log in over SSH with a key that can run
    only `bin/courier-deploy.sh` (it is part of the compose bundle, so it is on the server after the first deploy that has it).
    One-time setup:
    1. On your computer create a key pair used only for this: `ssh-keygen -t ed25519 -N '' -C courier-deploy -f courier-deploy`.
    2. On the server append the public key to `~/.ssh/authorized_keys` of user `occpl`, as one line with the restriction in front:
       `command="/opt/occpl/bin/courier-deploy.sh",restrict ssh-ed25519 AAAA... courier-deploy`
       (`restrict` turns off shells, forwarding and everything else; the script accepts only `backend|frontend <git SHA or dev>`).
    3. In **each** of the two courier repositories (Settings, Environments, new environment **dev**): secret `VPS_SSH_KEY` = the
       private key file, secret `VPS_KNOWN_HOSTS` = the output of `ssh-keyscan -t ed25519 <server address>`, variable `VPS_HOST` =
       the server address, variable `COURIER_URL` = `https://courier.dev.occplsoftech.com`. Delete the private key file afterwards.
    4. The server must be logged in to `ghcr.io` with the `GHCR_PULL_TOKEN` token (every OCCPL deploy does this); the courier
       deploy only runs `docker compose pull` for its own image.
    The image tags are kept in `.env` as `COURIER_BACKEND_RELEASE` and `COURIER_FRONTEND_RELEASE` (default `dev`), so the two
    images are released and rolled back independently. After the backend is replaced the web container is restarted, because its
    nginx looks the backend up only once at start.