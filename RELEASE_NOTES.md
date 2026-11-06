# WordPress SYNC 1.0.0

Initial public manual deployment release.

## Container

- Image: `ghcr.io/technicull-solutions/wordpress-sync:1.0.0`
- Digest: `sha256:4bfb25346a7a9d1afacfb9715a262264989307b85029bde1dafa41cf8b85c0a7`
- Created: `2026-10-03T16:34:09.587119703Z`
- Platform: `linux/amd64`

The versioned image tag is intended to remain immutable.

## Release assets

- `compose.yaml`
- `.env.example`
- `INSTALL.md`
- `RELEASE_NOTES.md`
- `SHA256SUMS`

## Included capabilities

- GitHub App access to private source repositories
- GitHub OAuth portal authentication
- Owner, operator, and read-only roles
- Theme and plugin deployment targets
- Local archive, SSH archive, and rsync deployment modes
- Repository preflight and dry-run review
- Backup and rollback controls
- External audit export
- Optional email notifications
- Optional controlled WordPress migration

## Deployment defaults

- The portal binds to `127.0.0.1:4177`.
- Public access requires an HTTPS reverse proxy.
- Real deployment remains disabled until explicitly enabled per target.
- Migration and email notifications are disabled by default.
- The GitHub App private key is mounted read-only.
- Persistent data remains outside the container.

## Security

The public release contains no operational credentials.

Each deployment supplies its own session secret, GitHub App credentials, OAuth
credentials, webhook secret, allowed users, and optional SMTP or migration
credentials.

Do not commit `.env`, private keys, runtime data, audit exports, database
credentials, or SMTP credentials.

## Acceptance requirements

Before enabling real deployment:

1. Verify the container digest.
2. Confirm container health.
3. Confirm GitHub App repository access.
4. Complete repository preflight.
5. Review a dry deployment.
6. Test backup and rollback on non-production.
7. Complete the MSP handoff record.
