---
longform:
  format: single
  title: Setup
title: Setup
---
# SonarQube with Docker Compose

This Docker Compose setup deploys a complete SonarQube environment with PostgreSQL database and SonarScanner CLI.

## Prerequisites

- Docker Engine 20.10.0+
- Docker Compose 2.0.0+
- At least 4GB of RAM available for Docker

## Architecture

This setup includes three services:

1. **PostgreSQL Database** (`sonarqube-db`) - Database backend for SonarQube
2. **SonarQube Server** (`sonarqube`) - Main SonarQube application
3. **SonarScanner CLI** (`sonar-scanner`) - Container for code analysis (runs on demand)

## Quick Start

1. **Clone or create the project directory:**

```bash
mkdir sonarqube-docker && cd sonarqube-docker
```

2. **Create the `docker-compose.yml` file:**

```yaml
version: '3.8'

services:
  sonarqube-db:
    image: postgres:15-alpine
    container_name: sonarqube-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonarqube
      POSTGRES_PASSWORD: sonarpass123
      POSTGRES_DB: sonarqube
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sonarqube"]
      interval: 10s
      timeout: 5s
      retries: 5

  sonarqube:
    image: sonarqube:latest
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      sonarqube-db:
        condition: service_healthy
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonarqube-db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonarqube
      SONAR_JDBC_PASSWORD: sonarpass123
      # Optional: Increase memory for better performance
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms128m"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
      # Mount source code directory for scanning (optional)
      - ./projects:/projects
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535

  sonar-scanner:
    image: sonarsource/sonar-scanner-cli:latest
    container_name: sonar-scanner-cli
    working_dir: /usr/src
    volumes:
      - ./projects:/usr/src
      - ./sonar-scanner/conf:/opt/sonar-scanner/conf
    networks:
      - sonarnet
    # This container doesn't run continuously
    command: ["sleep", "infinity"]

networks:
  sonarnet:
    driver: bridge

volumes:
  postgres_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
```

3. **Create required directories:**

```bash
mkdir -p projects sonar-scanner/conf
```

4. **Start the services:**

```bash
docker-compose up -d
```

## Accessing SonarQube

- **Web Interface:** http://localhost:9000
- **Default Credentials:** admin/admin (you'll be prompted to change on first login)

## Using SonarScanner

To analyze your code with the SonarScanner container:

1. **Place your project code in the `projects` directory**

2. **Create a `sonar-project.properties` file in your project root:**

```properties
# Configure here general information about the environment, such as server connection details for download of plugins
sonar.host.url=http://sonarqube:9000
# Default source code encoding

sonar.sourceEncoding=UTF-8
# Enable or disable the reporting of Issues which are automatically resolved when a line of code receives an update.

sonar.issuesReport.console.enable=true
# Security - using admin token is recommended

# sonar.login=your-generated-token-from-sonarqube-web-ui
# Optional: Java scanner specific

sonar.java.source=11
sonar.java.target=11

# Optional: For multi-language projects

sonar.language=java,js,py,ts
```

3. **Run the scanner for a specific project:**

```bash
# Execute scanner inside the container
docker exec sonar-scanner-cli sonar-scanner \
  -Dsonar.projectKey=my-project \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=your-auth-token
```

Or run it directly from your host machine:

```bash
docker run --rm \
  --network sonarqube-docker_sonarnet \
  -v "$(pwd)/projects:/usr/src" \
  sonarsource/sonar-scanner-cli:latest \
  sonar-scanner \
  -Dsonar.projectKey=my-project \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=your-auth-token
```

## Generating Authentication Token

1. Log into SonarQube at http://localhost:9000
2. Click your user icon → "My Account" → "Security"
3. Generate a token with a name like "scanner-token"
4. Use this token in your `sonar-project.properties` file or scanner command

## Maintenance Commands

```bash
# View logs
docker-compose logs -f sonarqube
docker-compose logs -f sonarqube-db

# Stop services
docker-compose stop

# Start services
docker-compose start

# Restart services
docker-compose restart

# Stop and remove containers, networks
docker-compose down

# Stop and remove containers, networks, and volumes
docker-compose down -v

# Check service status
docker-compose ps

# View resource usage
docker-compose stats
```