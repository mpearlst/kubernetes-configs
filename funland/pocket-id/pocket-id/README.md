# Pocket ID

Pocket ID is a simple and secure identity provider for self-hosted environments.

## Configuration

The deployment includes:

- **Namespace**: `pocket-id`
- **Port**: 1411 (exposed as port 80 on the service)
- **Storage**: 100m persistent volume for application data mounted at `/app/data`
- **Image**: `ghcr.io/pocket-id/pocket-id:v2`
- **Health Check**: Uses built-in healthcheck command

## Access

The application is accessible via HTTPRoute at:
- `https://id.batlab.io`

## Required Secrets

The deployment requires an `ENCRYPTION_KEY` stored in 1Password:

1. Create a new item in 1Password named `pocket-id-secrets`
2. Add a field named `encryption_key` with a value of at least 16 bytes (recommend 32+ character random string)
3. The ExternalSecret will automatically sync this to the cluster

The encryption key is used by Pocket ID to encrypt sensitive data.

## Resources

- **GitHub**: https://github.com/pocket-id/pocket-id
- **Docker Image**: https://github.com/pocket-id/pocket-id/pkgs/container/pocket-id
