# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains Docker Compose configurations for running various SonarQube versions alongside supporting services (PostgreSQL, LDAP, observability tools). It's designed for development and testing environments with different SonarQube deployments.

## System Requirements

SonarQube uses an embedded Elasticsearch that requires specific kernel parameters. On Rancher Desktop, set these before starting any SonarQube service:

```bash
# Run an interactive shell into the VM
rdctl shell

# Set required kernel parameters
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
```

Reference: https://hub.docker.com/_/sonarqube

## Common Commands

### Starting Services

Navigate to the directory of interest and run:
```bash
docker-compose up
```

For background execution:
```bash
docker-compose up -d
```

### Stopping Services

```bash
docker-compose down
```

To remove volumes as well:
```bash
docker-compose down -v
```

### Viewing Logs

```bash
docker-compose logs -f [service-name]
```

## Architecture

The repository is organized by service type with each directory containing a complete Docker Compose stack:

### SonarQube Environments

Three different SonarQube versions are available, each with its own complete stack:

- **sonarqube-latest/**: Latest enterprise version (2025.4.2) with nginx reverse proxy, SSL/TLS support, ngrok tunneling, and VictoriaMetrics monitoring
- **sonarqube-9/**: Enterprise version 9.9.9 LTA with PostgreSQL backend, MCP server integration, ngrok tunneling, and metrics collection
- **sonarqube-2025.1/**: Enterprise version 2025.1 (identical setup to sonarqube-9)

#### SonarQube Stack Components

Each SonarQube directory follows a common pattern:

**Core Services:**
- SonarQube application server (exposed on different ports per version)
- Configuration, data, extensions, and logs mounted from host directories

**Supporting Services:**
- PostgreSQL database (sonarqube-9 and sonarqube-2025.1 only; sonarqube-latest uses embedded DB)
- nginx reverse proxy (sonarqube-latest only) - handles HTTP/HTTPS with custom SSL certificates
- ngrok tunnel service (all versions) - exposes SonarQube via public URL for webhooks/integrations
- VictoriaMetrics agent (all versions) - scrapes metrics and forwards to central VictoriaMetrics instance on host

**Port Mappings:**
- sonarqube-latest: 9010 (HTTP), 9011 (HTTPS), 4040 (ngrok dashboard)
- sonarqube-9: 19000 (SonarQube), 4040 (ngrok dashboard), 8080 (MCP server)
- sonarqube-2025.1: 19000 (SonarQube), 4040 (ngrok dashboard), 8080 (MCP server)

### SSL/TLS Configuration for sonarqube-latest

The sonarqube-latest deployment requires manual certificate generation as it uses nginx with HTTPS. Follow these steps in the `sonarqube-latest/` directory:

1. Generate CA private key and certificate:
```bash
openssl req -x509 -new -nodes -keyout ca.key -days 3650 -out ca.crt
```

2. Generate server private key and CSR:
```bash
openssl genpkey -algorithm RSA -out server.key
openssl req -new -key server.key -out server.csr
```

3. Create `server.ext` file with Subject Alternative Names matching your FQDN

4. Sign the server certificate:
```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -extfile server.ext
```

The certificates are mounted into the nginx container via docker-compose volumes.

### MCP Server Integration

Both sonarqube-9 and sonarqube-2025.1 include an MCP (Model Context Protocol) server container that connects to SonarQube:

```yaml
sonarqube-9-mcp:
  image: mcp/sonarqube:latest
  environment:
    SONARQUBE_TOKEN: ${SONAR_TOKEN}
    SONARQUBE_URL: ${SONAR_HOST_URL}
```

Requires environment variables `SONAR_TOKEN` and `SONAR_HOST_URL` to be set.

### Supporting Services

- **observability/**: VictoriaMetrics and Grafana stack for monitoring metrics from SonarQube instances
  - VictoriaMetrics runs on host network mode (port 8428)
  - Grafana accessible on port 3000
  - vmagent scrapes metrics and forwards to VictoriaMetrics

- **postgres/**: Standalone PostgreSQL 17 instance on host network mode
  - Used for general database needs
  - Default credentials: postgres/123456

- **ldap/**: OpenLDAP server with phpLDAPadmin web UI
  - OpenLDAP on host network mode (default admin: cn=admin,dc=example,dc=org)
  - phpLDAPadmin accessible on port 7443

- **hello-world/**: Simple test container to verify Docker Compose setup

## Environment Variables

Several services require environment variables. Create a `.env` file in the relevant directory:

```bash
# For ngrok tunneling (all SonarQube instances)
NGROK_AUTHTOKEN=your_ngrok_token

# For MCP server (sonarqube-9 and sonarqube-2025.1)
SONAR_TOKEN=your_sonarqube_token
SONAR_HOST_URL=your_sonarqube_url

# For PostgreSQL (sonarqube-9 and sonarqube-2025.1)
POSTGRES_PASSWORD=your_postgres_password
```

## Networking Architecture

Each SonarQube deployment uses isolated Docker networks:

- **Backend networks** (e.g., `sonarqube-latest-backend`): Internal communication between SonarQube, database, monitoring agents
- **Frontend networks** (e.g., `sonarqube-latest-frontend`): Exposed via nginx/reverse proxy
- **Host network mode**: Used by observability, postgres, and ldap services for direct host access

Services use `host.docker.internal` to communicate with services on the Docker host (e.g., vmagent forwarding metrics to VictoriaMetrics).

## CI/CD Integration

GitHub Actions workflow (`.github/workflows/build.yml`) runs SonarQube analysis on PRs and pushes to main:

```yaml
- SonarQube scan using sonarqube-scan-action@v6
- Requires secrets: SONAR_TOKEN
- Requires vars: SONAR_HOST_URL
- Project configuration in sonar-project.properties
```

Project: `waynekhan-net_MAC0432-docker` in organization `github-com-waynekhan-net`

## Volume Management

Persistent data is stored in Docker named volumes:
- SonarQube: conf/, data/, extensions/, logs/ (bind mounts)
- PostgreSQL: `var-lib-postgresql-data`
- Grafana: `var-lib-grafana`
- VictoriaMetrics: `storage` and `vmagentdata`

To inspect volumes:
```bash
docker volume ls
docker volume inspect <volume-name>
```

## Working with Multiple SonarQube Versions

When working with multiple SonarQube versions simultaneously:
1. Each version has unique container names and ports to avoid conflicts
2. Navigate to the specific directory before running docker-compose commands
3. Use `docker-compose ps` to check running services
4. Remember to set kernel parameters before starting any SonarQube instance
