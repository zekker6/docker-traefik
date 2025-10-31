# Docker Traefik Setup

This repository contains a Docker Compose setup for running [Traefik](https://traefik.io/) as a reverse proxy with automatic service discovery and host file generation.

## Prerequisites

- Docker
- Docker Compose

## Quick Start

### 1. Create the Docker Network

Before starting the services, you need to create an external Docker network called `tk_web`:

```bash
docker network create tk_web
```

This network is used by Traefik to communicate with other services and must be created before running `docker compose up`.

### 2. Start the Services

Start Traefik and the hosts generator:

```bash
docker compose up -d
```

This will start two services:
- **traefik**: The Traefik reverse proxy (v3.5.4)
- **tk-hosts**: A hosts file generator that automatically updates `/etc/hosts` based on discovered services

### 3. Access the Traefik Dashboard

Once the services are running, you can access the Traefik dashboard at:

```
http://localhost:8080
```

The dashboard provides an overview of all services, routers, and middlewares managed by Traefik.

## Connecting Other Services

To connect other Docker services to this Traefik instance, you need to:

### 1. Add the Service to the `tk_web` Network

In your service's `docker-compose.yml`, add the `tk_web` network:

```yaml
version: '3'

services:
  myapp:
    image: myapp:latest
    networks:
      - tk_web

networks:
  tk_web:
    external: true
```

### 2. Configure Traefik Labels (Optional)

By default, Traefik will automatically create a route for your service using the naming convention:

```
Host(`service-name.docker.loc`)
```

For example, a container named `my-web-app` would be accessible at `http://my.web.app.docker.loc`.

You can override this behavior with Traefik labels:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - tk_web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.local`)"
      - "traefik.http.routers.myapp.entrypoints=web"
      - "traefik.http.services.myapp.loadbalancer.server.port=80"
```

### 3. Access Your Service

If using the automatic host file generator, services will be accessible via their generated hostname (e.g., `http://service-name.docker.loc`).

Otherwise, you can access services through:
- **Port 80**: HTTP traffic routed by Traefik
- Direct access via localhost if you've configured custom Host rules

## Configuration

### Traefik Configuration (`traefik.yml`)

The Traefik configuration includes:

- **Docker Provider**: Automatically discovers services running on Docker
- **Default Rule**: `Host('{{range $i, $e := splitList "-" .Name}}{{e}}.{{end}}docker.loc')`
  - Converts container names to hostnames (e.g., `my-service` → `my.service.docker.loc`)
- **Network**: Uses the `tk_web` network for service discovery
- **API**: Insecure API enabled for the dashboard

### Hosts Generator

The `tk-hosts` service automatically updates `/etc/hosts` on the host machine with entries for all discovered services. This allows you to access services using their generated domain names without additional DNS configuration.

**Note**: This requires mounting `/etc/hosts` as a volume and appropriate permissions on the host.

## Ports

- **80**: HTTP entrypoint for all routed services
- **8080**: Traefik dashboard and API

## Stopping the Services

To stop the services:

```bash
docker compose down
```

To stop and remove volumes:

```bash
docker compose down -v
```

## Troubleshooting

### Network not found

If you see an error about the `tk_web` network not being found, make sure you've created it:

```bash
docker network create tk_web
```

### Services not appearing in Traefik

1. Ensure your service is connected to the `tk_web` network
2. Check that the service container is running: `docker ps`
3. Verify Traefik logs: `docker compose logs traefik`
4. Ensure Traefik can access the Docker socket

### Cannot access the dashboard

- Verify Traefik is running: `docker compose ps`
- Check that port 8080 is not in use by another application
- Check Traefik logs for errors: `docker compose logs traefik`

## License

This project configuration is provided as-is for use with Traefik.
