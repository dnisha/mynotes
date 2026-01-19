---
longform:
  format: single
  title: Trivy
title: Trivy
---
# Trivy: Security Scanner

## Overview
Trivy is a **comprehensive security scanner** for containers and other artifacts that detects:
- Vulnerabilities in OS packages (Alpine, RHEL, Debian, Ubuntu, etc.)
- Language-specific packages (Bundler, Composer, npm, yarn, Cargo, etc.)
- **Misconfigurations** (Kubernetes, Docker, Terraform, AWS, etc.)
- **Secrets** (API keys, passwords, tokens)
- **SBOM generation** (Software Bill of Materials)
- **License compliance**

## Key Features

### 1. **Multi-Purpose Scanner**
- **Vulnerability Scanner**: OS packages + application dependencies
- **Misconfiguration Scanner**: IaC, Kubernetes, Dockerfiles
- **Secret Scanner**: Detect exposed credentials

### 2. **Simple & Comprehensive**
- Single binary, no dependencies
- No pre-requisites (DB, libraries, etc.)
- Quick first scan (under 10 sec for most images)

### 3. **CI/CD Friendly**
- Easy to integrate in pipelines
- JSON, SARIF, template outputs
- Exit codes for CI failures

## Architecture

```
┌─────────────────────────────────────────┐
│           Trivy Client                  │
│  ┌─────────┬─────────┬──────────────┐  │
│  │ Scanner │  Cache  │   Output     │  │
│  │ Engine  │  Layer  │  Formatter   │  │
│  └─────────┴─────────┴──────────────┘  │
└───────────────────┬─────────────────────┘
                    │
    ┌───────────────┼───────────────┐
    │     Local     │    Remote     │
    │   Database    │   Registry    │
    │   (Vuln DB)   │   (Images)    │
    └───────────────┴───────────────┘
```

### Components:
1. **Scanner Engine**: Detects vulnerabilities/misconfigs
2. **Cache Layer**: Stores vulnerability DB locally
3. **Output Formatter**: Multiple output formats
4. **Vulnerability DB**: Built-in, auto-updated

## Common Commands & Flags

### Basic Scanning
```bash
# Scan container image
trivy image <image_name>

# Scan filesystem
trivy fs <directory>

# Scan repository
trivy repo <repo_url>

# Scan Kubernetes cluster
trivy k8s --report summary all
```

### Output Formats
```bash
trivy image --format table alpine:latest      # Default
trivy image --format json alpine:latest       # JSON
trivy image --format template alpine:latest   # Custom Go template
trivy image --format sarif alpine:latest      # SARIF for GitHub
```

### Scan Types
```bash
# Specific scanners
trivy image --scanners vuln,misconfig,secret <image>
trivy config .        # IaC misconfigurations
trivy rootfs /        # Root filesystem
```

### Severity & Filtering
```bash
# Filter by severity
trivy image --severity HIGH,CRITICAL alpine:latest

# Ignore specific vulnerabilities
trivy image --ignore-unfixed alpine:latest
trivy image --ignorefile .trivyignore alpine:latest

# Limit results
trivy image --limit 10 alpine:latest
```

### Cache Management
```bash
# Clear cache
trivy --clear-cache

# Skip update
trivy image --skip-db-update alpine:latest

# Download DB only
trivy --download-db-only
```

### CI/CD Integration
```bash
# Exit with code on findings
trivy image --exit-code 1 alpine:latest

# Scan with timeout
trivy image --timeout 5m alpine:latest

# Quiet mode
trivy image --quiet alpine:latest
```

### Registry Authentication
```bash
# Private registry
trivy image --username <user> --password <pass> private.reg/image

# Docker config
trivy image --input <docker-save.tar>
```

## Practical Examples

### 1. **Quick Container Scan**
```bash
trivy image ubuntu:latest
```

### 2. **CI Pipeline Scan**
```bash
trivy image \
  --format sarif \
  --output results.sarif \
  --exit-code 1 \
  --severity CRITICAL,HIGH \
  myapp:latest
```

### 3. **Multi-target Scan**
```bash
# Scan multiple images
trivy image --input images.txt

# Scan directory with specific scanners
trivy fs --scanners misconfig,secret /path/to/iac
```

### 4. **SBOM Generation**
```bash
# Generate CycloneDX SBOM
trivy image --format cyclonedx myapp:latest

# Output to file
trivy image --format spdx-json --output sbom.spdx.json myapp:latest
```

## Configuration File
Create `.trivy.yaml`:
```yaml
db:
  skip-update: true
cache:
  dir: "/custom/cache"
misconfiguration:
  include:
    - "dockerfile"
    - "kubernetes"
severity:
  - CRITICAL
  - HIGH
```

## Integration Points

- **GitHub Actions**: `aquasecurity/trivy-action`
- **GitLab CI**: `trivy` template
- **Jenkins**: `trivy` plugin
- **Kubernetes**: Operator or admission controller
- **Harbor**: Built-in integration

## Performance Tips

1. **Use Cache**: Default location `~/.cache/trivy`
2. **Skip DB Update** in CI if recent scan
3. **Limit Scanners** to only what you need
4. **Use Severity Filters** to reduce noise
5. **Scan Slim Images** for faster results

## Comparison Advantage

| Feature | Trivy | Alternatives |
|---------|-------|-------------|
| Ease of Use | Single binary | Complex setups |
| Speed | <10s first scan | Minutes for first scan |
| Coverage | Vuln+Config+Secrets+SBOM | Usually single-purpose |
| Maintenance | Self-contained | External DB required |

Trivy's **all-in-one** approach makes it ideal for **DevSecOps pipelines** where simplicity, speed, and comprehensive coverage are critical.