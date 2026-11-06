# Install WordPress SYNC 1.0.0 on Ubuntu

This release installs WordPress SYNC on headless Ubuntu using Docker Engine and
Docker Compose.

```text
Image: ghcr.io/technicull-solutions/wordpress-sync:1.0.0
Digest: sha256:4bfb25346a7a9d1afacfb9715a262264989307b85029bde1dafa41cf8b85c0a7
Platform: linux/amd64
```

The deployment bundle and container image are public. No GitHub credential is
required to download or pull them.

Runtime access to private WordPress source repositories uses a separately
configured GitHub App.

## Prerequisites

Obtain/record before installation:

- The public HTTPS hostname for WordPress SYNC
- The existing WordPress `wp-content` and `themes` host paths
- GitHub App ID and private key
- GitHub OAuth client ID and secret
- GitHub webhook secret
- GitHub usernames and assigned portal roles


## 1. Install Docker Engine and Compose

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

. /etc/os-release

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${UBUNTU_CODENAME:-$VERSION_CODENAME} stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable --now docker
```

Verify:

```bash
systemctl is-enabled docker
systemctl is-active docker
sudo /usr/bin/docker --version
sudo /usr/bin/docker compose version
sudo /usr/bin/docker run --rm hello-world
```

Docker must report `enabled` and `active`. The test container must display
`Hello from Docker!`.

## 2. Download and verify the release

```bash
sudo install -d \
  -m 0750 \
  -o "$USER" \
  -g "$USER" \
  /opt/wordpress-sync

cd /opt/wordpress-sync

RELEASE_URL="https://github.com/Technicull-Solutions/wordpress-sync-deploy/releases/download/v1.0.0"

for asset in \
  compose.yaml \
  .env.example \
  INSTALL.md \
  RELEASE_NOTES.md \
  SHA256SUMS
do
  curl -fL \
    "${RELEASE_URL}/${asset}" \
    -o "${asset}"
done

sha256sum --check SHA256SUMS
```

Every file must report `OK`. Stop if verification fails.

## 3. Prepare persistent storage

Set the existing WordPress paths:

```bash
WORDPRESS_CONTENT_PATH="/absolute/path/to/wp-content"
WORDPRESS_THEMES_PATH="${WORDPRESS_CONTENT_PATH}/themes"
```

Confirm they exist:

```bash
test -d "$WORDPRESS_CONTENT_PATH"
test -d "$WORDPRESS_THEMES_PATH"

stat \
  -c 'Path: %n
Owner UID: %u
Owner GID: %g
Permissions: %A' \
  "$WORDPRESS_CONTENT_PATH" \
  "$WORDPRESS_THEMES_PATH"
```

Choose a UID and GID that can write only the required WordPress paths. Record
those values as `SYNC_UID` and `SYNC_GID`.

Create the persistent directories using that identity:

```bash
SYNC_UID_VALUE="REPLACE_WITH_UID"
SYNC_GID_VALUE="REPLACE_WITH_GID"

cd /opt/wordpress-sync

sudo install -d \
  -m 0700 \
  -o "$SYNC_UID_VALUE" \
  -g "$SYNC_GID_VALUE" \
  secrets

sudo install -d \
  -m 0750 \
  -o "$SYNC_UID_VALUE" \
  -g "$SYNC_GID_VALUE" \
  data \
  audit-export \
  backups \
  imports \
  wordpress-site \
  wordpress-themes \
  wordpress-content
```

Persistent data must remain outside the replaceable container.

## 4. Configure `.env`

```bash
cd /opt/wordpress-sync

cp .env.example .env
chmod 0600 .env
```

Generate independent session and webhook secrets:

```bash
openssl rand -hex 32
openssl rand -hex 32
```

Edit the environment file:

```bash
nano .env
```

At minimum, configure:

```env
SYNC_IMAGE=ghcr.io/technicull-solutions/wordpress-sync:1.0.0
SYNC_PULL_POLICY=always

SYNC_BASE_URL=https://REPLACE_WITH_SYNC_HOSTNAME
SYNC_BIND_IP=127.0.0.1
SYNC_PORT=4177

SYNC_UID=REPLACE_WITH_UID
SYNC_GID=REPLACE_WITH_GID

WORDPRESS_CONTENT_HOST_PATH=/absolute/path/to/wp-content
WORDPRESS_THEMES_HOST_PATH=/absolute/path/to/wp-content/themes

SYNC_SESSION_SECRET=REPLACE_WITH_SESSION_SECRET

GITHUB_APP_ID=REPLACE_WITH_NUMERIC_APP_ID
GITHUB_APP_PRIVATE_KEY_PATH=/app/secrets/github-app-private-key.pem
GITHUB_OAUTH_CLIENT_ID=REPLACE_WITH_OAUTH_CLIENT_ID
GITHUB_OAUTH_CLIENT_SECRET=REPLACE_WITH_OAUTH_CLIENT_SECRET
GITHUB_WEBHOOK_SECRET=REPLACE_WITH_WEBHOOK_SECRET
SYNC_GITHUB_USERS=github-login:owner

SYNC_MIGRATION_ENABLED=false
SYNC_EMAIL_NOTIFICATIONS_ENABLED=false
```

Keep `SYNC_ADMIN_PASSWORD` commented out when GitHub OAuth is configured.

Review remaining placeholders:

```bash
grep -nEi \
  'replace-with|replace_with|example\.com|example-org|example-wordpress' \
  .env
```

Resolve every applicable result before starting the service.

## 5. Install the GitHub App private key

Transfer the PEM file through the approved secure-transfer system.

```bash
cd /opt/wordpress-sync

SYNC_UID_VALUE="$(sed -n 's/^SYNC_UID=//p' .env)"
SYNC_GID_VALUE="$(sed -n 's/^SYNC_GID=//p' .env)"

sudo install \
  -m 0600 \
  -o "$SYNC_UID_VALUE" \
  -g "$SYNC_GID_VALUE" \
  /path/to/transferred/github-app-private-key.pem \
  secrets/github-app-private-key.pem
```

Verify permissions without displaying the key:

```bash
stat \
  -c '%a %u:%g %n' \
  secrets/github-app-private-key.pem
```

Expected mode:

```text
600
```

## 6. Validate Compose and pull the image

```bash
cd /opt/wordpress-sync

sudo /usr/bin/docker compose config --quiet
sudo /usr/bin/docker compose pull sync
```

No registry login is required.

Verify the downloaded image:

```bash
sudo /usr/bin/docker image inspect \
  ghcr.io/technicull-solutions/wordpress-sync:1.0.0 \
  --format '{{index .RepoDigests 0}}'
```

Expected result:

```text
ghcr.io/technicull-solutions/wordpress-sync@sha256:4bfb25346a7a9d1afacfb9715a262264989307b85029bde1dafa41cf8b85c0a7
```

Stop if the digest differs.

## 7. Start WordPress SYNC

```bash
cd /opt/wordpress-sync

sudo /usr/bin/docker compose up -d sync
sudo /usr/bin/docker compose ps
```

Wait for `healthy`, then test locally:

```bash
curl -fsS http://127.0.0.1:4177/healthz
```

Review startup logs if required:

```bash
sudo /usr/bin/docker compose logs \
  --tail 100 \
  sync
```

## 8. Publish through HTTPS

Keep the service bound to `127.0.0.1`. Configure the existing reverse proxy to
forward the approved HTTPS hostname to:

```text
http://127.0.0.1:4177
```

If an identity-aware proxy protects the portal, bypass only:

```text
/api/webhook/github
```

WordPress SYNC still verifies the GitHub webhook signature.

## 9. Configure and validate the portal

Sign in with an allowed GitHub account and configure:

1. Branding and portal links
2. Repository profiles
3. The private `canfasd-iad-hub` repository
4. Development, staging, and production targets
5. Theme and plugin source paths
6. Mounted WordPress destination paths
7. User roles
8. Audit and backup settings

Inside the container, the local destinations are:

```text
/wordpress-content
/wordpress-themes
```

Keep automatic deployment and **Allow real deploys** disabled.

Run the readiness report:

```bash
sudo /usr/bin/docker compose exec \
  sync \
  npm run check:config
```

Then:

1. Confirm the GitHub App can read `canfasd-iad-hub`.
2. Run repository preflight.
3. Run a dry deployment.
4. Review every proposed change.
5. Verify the dry run changed no WordPress files.
6. Verify settings survive a container restart.
7. Test backup and rollback on non-production before enabling real deployment.

