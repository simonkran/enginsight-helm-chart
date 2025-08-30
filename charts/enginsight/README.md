# enginsight

Enginsight Enterprise - Comprehensive security and compliance platform

![Version: 1.4.0](https://img.shields.io/badge/Version-1.4.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 7.8.11](https://img.shields.io/badge/AppVersion-7.8.11-informational?style=flat-square)

## Installation

```bash
helm install enginsight oci://ghcr.io/simonkran/charts/enginsight
# helm -n enginsight install enginsight oci://ghcr.io/simonkran/charts/enginsight --create-namespace
# helm -n enginsight upgrade --install enginsight oci://ghcr.io/simonkran/charts/enginsight -f values.yaml
```

## Configuration

### Add private container registry credentials
```bash
kubectl create secret docker-registry enginsight-registry-secret \
  --docker-server=registry.enginsight.com \
  --docker-username=username \
  --docker-password=password \
  --namespace=enginsight
```

### Create required Enginsight Services config.json as a Kubernetes Secret

| Placeholder | Example |
|-------------|--------------|
| `%%APP_URL%%` | https://ui-enginsight.example.com |
| `%%API_URL%%` | https://api-enginsight.example.com |
| `%%JWT_SECRET%%` | provide custom generated JWT |
| `%%MONGODB_URI%%` | mongodb://enginsight-mongodb-headless.enginsight.svc.cluster.local:27017/enginsight?replicaSet=rs0 |
| `%%MONGODB_CVES_URI%%` | mongodb://enginsight-mongodb-cves-headless.enginsight.svc.cluster.local:27017/cves |
| `%%REDIS_MASTER%%` | redis://enginsight-redis-master.enginsight.svc.cluster.local:6379 |

Replace placeholders with actual values.

```bash
# config.json
{
  "onpremise": {
    "twoFactor": {
      "enabled": false
    }
  },
  "api": {
    "url": "%%API_URL%%",
    "port": 8080,
    "timeout": 5000,
    "clientMaxBodySize": "50mb",
    "cookies": {
      "maxAge": 3600000,
      "domain": "",
      "path": "/",
      "signed": false
    },
    "recaptcha": {
      "secret": "",
      "host": "www.google.com",
      "path": "/recaptcha/api/siteverify",
      "method": "POST"
    },
    "headers": {},
    "jwt": {
      "secret": "%%JWT_SECRET%%",
      "expiresIn": "24h"
    },
    "cors": {
          "origin": "%%APP_URL%%",
          "credentials": true
    }
  },
  "loopDelay": 15000,
  "database": {
    "uriConnectionString": "%%MONGODB_URI%%"
  },
  "cache": "%%REDIS_MASTER%%",
  "email": {
    "sender": "'Enginsight' <no-reply@your.domain>",
    "host": "",
    "port": "",
    "sslTls": true,
    "user": "",
    "pass": "",
    "maxConnections": 2,
    "rateDelta": 1000,
    "rateLimit": 1
  },
  "integrations": {
    "slack": {
      "clientId": "",
      "clientSecret": ""
    }
  },
  "cves": {
    "uriConnectionString": "%%MONGODB_CVES_URI%%"
  },
  "app": {
    "host": "%%APP_URL%%"
  },
  "sms": {
    "limit": 20,
    "region": "",
    "accessKeyId": "",
    "secretAccessKey": "",
    "senderId": ""
  }
}
```

Create a Kubernetes Secret.
```bash
kubectl -n namespace create secret generic enginsight-services-config --from-file=config.json
```

### Configure Ingress

Enginsight requires the following components to be pre-installed in your cluster:

- An Ingress Controller (such as Traefik or ingress-nginx)
- cert-manager for TLS certificate management

Enginsight deploys with ingress enabled by default for both UI and API services. You'll need to configure your ingress settings to match your cluster environment.

```yaml
# values.yaml
services:
  ui:
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-http01-prod
      className: "traefik"  # or "nginx", "alb", etc.
      hosts:
        - host: ui-enginsight.example.com
          paths:
            - path: /
              pathType: ImplementationSpecific
      tls:
        - secretName: ui-enginsight.example.com
          hosts:
            - ui-enginsight.example.com
 
  server:
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-http01-prod
      className: "traefik"
      hosts:
        - host: api-enginsight.example.com
          paths:
            - path: /
              pathType: ImplementationSpecific
      tls:
        - secretName: api-enginsight.example.com
          hosts:
            - api-enginsight.example.com
```

### External Dependencies
Enginsight requires three backend services, mongodb, mongodb-cves and redis. If not externally provided or for development purposes, set the following properties to `true`: `mongodb.enabled`, `mongodb-cves.enabled` and `redis.enabled`.
