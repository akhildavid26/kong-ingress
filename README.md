# Kong API Gateway Security Tutorial
This repository contains both starter code and Kong security implementation for the Kong API Gateway Tutorial.

## Part 1: Basic Setup (Without Kong)

### 1. Apply the starter manifests:
```bash
kubectl apply -f starter-manifests.yaml
```

### 2. Test the API (Basic Setup):
```bash
# Add grades
curl -X POST http://localhost:31000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Harry", "subject": "Defense Against Dark Arts", "score": 95}'

curl -X POST http://localhost:31000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Ron", "subject": "Charms", "score": 82}'

curl -X POST http://localhost:31000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Hermione", "subject": "Potions", "score": 98}'

# Get all grades
curl http://localhost:31000/grades
```

## Part 2: Kong API Gateway Implementation

### 1. Kong Plugin Templates
Copy and customize these templates for Kong implementation:

```yaml
# kong-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grade-submission
  namespace: grade-demo
  annotations:
    konghq.com/strip-path: "false"
    konghq.com/plugins: grade-auth, grade-ratelimit    # Your plugin names
    kubernetes.io/ingress.class: kong
    konghq.com/methods: "GET, POST"
spec:
  ingressClassName: kong
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grade-submission-api    # Your service name
                port:
                  number: 3000               # Your service port

# kong-plugins.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: grade-ratelimit    # Referenced in Ingress annotations
  namespace: grade-demo
config:
  minute: 10              # Adjust rate limit as needed
  limit_by: consumer
  policy: local
plugin: rate-limiting
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: grade-auth        # Referenced in Ingress annotations
  namespace: grade-demo
config:
  key_names:
    - apikey             # API key header name
  hide_credentials: true
plugin: key-auth

# kong-consumer.yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: grade-submission-consumer    # Your consumer name
  namespace: grade-demo
username: grade-submission           # Your username
custom_id: grade-submission-consumer-1  # Your custom ID
credentials:
  - user1-apikey                    # Reference to Secret name

# kong-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: user1-apikey                # Referenced in KongConsumer
  namespace: grade-demo
  labels:
    konghq.com/credential: key-auth
type: Opaque
stringData:
  kongCredType: key-auth
  key: your-secret-key              # Replace with your desired API key
```

### 2. Test API with Kong Security:
```bash
# Replace <node-port> with Kong proxy port and your-secret-key with the key from kong-secret.yaml
# Add grades
curl -X POST http://localhost:<node-port>/grades \
  -H "apikey: your-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"name": "Harry", "subject": "Defense Against Dark Arts", "score": 95}'

# Get all grades
curl http://localhost:<node-port>/grades -H "apikey: your-secret-key"
```

### 3. Common Issues:
- Ensure Kong is properly installed in your cluster
- Verify all resources are in the same namespace
- Check Kong proxy port is correctly configured
- Verify API key matches the secret value
```