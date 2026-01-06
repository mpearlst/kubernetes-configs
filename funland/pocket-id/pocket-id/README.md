# Pocket ID

Pocket ID is a simple and secure identity provider for self-hosted environments.

## Configuration

The deployment includes:

- **Namespace**: `pocket-id`
- **Port**: 1411 (exposed as port 80 on the service)
- **Storage**: 5Gi persistent volume for application data mounted at `/app/data`
- **Image**: `ghcr.io/pocket-id/pocket-id:v2`
- **Health Check**: Uses built-in healthcheck command

## Access

The application is accessible via HTTPRoute at:
- `https://pocket-id.batlab.io`

## Environment Variables

If you need to configure environment variables, you can add a ConfigMap and reference it in the deployment. Common configuration options should be added based on the `.env` file requirements from the upstream project.

## Resources

- **GitHub**: https://github.com/pocket-id/pocket-id
- **Docker Image**: https://github.com/pocket-id/pocket-id/pkgs/container/pocket-id
