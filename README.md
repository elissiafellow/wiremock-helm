# WireMock Helm Chart - Localhost Testing

This directory contains the Helm values configuration for WireMock, aligned with the wiremock directory structure.

## Configuration Overview

The `values.yaml` has been configured to match the wiremock directory setup:

- **Port**: 8082 (NI API service)
- **Image**: `wiremock/wiremock:latest`
- **Mappings**: NI API mappings from `wiremock/configmaps/ni-mappings.yaml`
- **Responses**: NI API responses from `wiremock/configmaps/ni-responses.yaml`
- **Service Type**: NodePort (for localhost access)
- **Response Templating**: Global response templating enabled

## Quick Start

### Option 1: Using Helm (Recommended)

If you have a Helm chart available:

```bash
# Install WireMock
helm install wiremock . -n wiremock --create-namespace -f values.yaml

# Or use the test script
./test-localhost.sh
```

### Option 2: Using ArgoCD (GitOps)

Deploy WireMock using ArgoCD with the official Helm chart from [wiremock/helm-charts](https://github.com/wiremock/helm-charts):

```bash
# Apply the ArgoCD Application manifest
kubectl apply -f argocd-application.yaml

# Check application status
kubectl get application wiremock -n argocd

# View application details in ArgoCD UI
# Access ArgoCD UI and navigate to the wiremock application
```

The `argocd-application.yaml` file references:
- **Chart Repository**: `https://github.com/wiremock/helm-charts`
- **Chart Path**: `charts/wiremock`
- **Values**: Inline values matching your `values.yaml` configuration

**Alternative: Using Separate Values File**

If you prefer to store your values.yaml in a Git repository, use `argocd-application-with-values-file.yaml` instead. This requires:
- ArgoCD v2.6+
- Your values.yaml file stored in a Git repository
- Configure the `sources` section to reference both the chart repo and values repo

### Option 3: Using Direct kubectl (Alternative)

If you prefer to use the existing `wiremock-runner.yaml`:

```bash
# Apply ConfigMaps first
kubectl apply -f ../wiremock/configmaps/ni-mappings.yaml
kubectl apply -f ../wiremock/configmaps/ni-responses.yaml

# Apply the deployment
kubectl apply -f ../wiremock/wiremock-runner.yaml

# Port forward for localhost access
kubectl port-forward -n wiremock svc/wiremock-runner 8082:8082
```

## Testing on Localhost

### Method 1: Port Forwarding

```bash
# Forward port 8082
kubectl port-forward -n wiremock svc/wiremock-runner 8082:8082

# In another terminal, test the API
curl -X POST http://localhost:8082/api/v1/MFCard/getPersonInfoByMnfId \
  -H "Content-Type: application/json" \
  -d '{"mnfuuid": "1102065"}'
```

### Method 2: NodePort (if service type is NodePort)

```bash
# Get the NodePort
kubectl get svc -n wiremock wiremock-runner -o jsonpath='{.spec.ports[?(@.name=="ni-api")].nodePort}'

# Access via localhost (for minikube)
minikube service wiremock-runner -n wiremock --url

# Or directly (for kind/Docker Desktop)
# http://localhost:<NODEPORT>
```

### Method 3: Ingress (if enabled)

If you have an ingress controller running:

```bash
# Add to /etc/hosts (or C:\Windows\System32\drivers\etc\hosts on Windows)
127.0.0.1 wiremock.local

# Access via
curl http://wiremock.local/api/v1/MFCard/getPersonInfoByMnfId \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"mnfuuid": "1102065"}'
```

## API Endpoints

### NI API Service (Port 8082)

- **Health Check**: `GET http://localhost:8082/__admin/health`
- **Mappings**: `GET http://localhost:8082/__admin/mappings`
- **NI API Endpoint**: `POST http://localhost:8082/api/v1/MFCard/getPersonInfoByMnfId`

### Test Requests

**Success Case:**
```bash
curl -X POST http://localhost:8082/api/v1/MFCard/getPersonInfoByMnfId \
  -H "Content-Type: application/json" \
  -d '{"mnfuuid": "1102065"}'
```

**Error Case (Empty UUID):**
```bash
curl -X POST http://localhost:8082/api/v1/MFCard/getPersonInfoByMnfId \
  -H "Content-Type: application/json" \
  -d '{"mnfuuid": ""}'
```

## Configuration Details

### Mappings

The mappings are configured in `values.yaml` under the `mappings` section:
- `getPersonInfoByMnfId-success.json` - Handles requests with valid mnfuuid
- `getPersonInfoByMnfId-empty-uuid.json` - Handles requests with empty mnfuuid

### Responses

The responses are configured in `values.yaml` under the `responses` section:
- `getPersonInfoByMnfId-success.json` - Success response with person data
- `getPersonInfoByMnfId-empty-uuid.json` - Error response for empty UUID

## Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl get pods -n wiremock

# Check logs
kubectl logs -n wiremock -l app.kubernetes.io/name=wiremock

# Describe pod for events
kubectl describe pod -n wiremock -l app.kubernetes.io/name=wiremock
```

### ConfigMaps not mounted

```bash
# Verify ConfigMaps exist
kubectl get configmaps -n wiremock

# Check volume mounts
kubectl describe pod -n wiremock -l app.kubernetes.io/name=wiremock | grep -A 10 "Mounts:"
```

### Port forwarding issues

```bash
# Check if port is already in use
lsof -i :8082

# Kill existing port forward
pkill -f "kubectl port-forward"

# Restart port forward
kubectl port-forward -n wiremock svc/wiremock-runner 8082:8082
```

## Cleanup

```bash
# Remove ArgoCD Application (if deployed via ArgoCD)
kubectl delete application wiremock -n argocd

# Remove Helm release (if deployed via Helm)
helm uninstall wiremock -n wiremock

# Or remove kubectl resources (if deployed directly)
kubectl delete -f ../wiremock/wiremock-runner.yaml
kubectl delete -f ../wiremock/configmaps/ni-mappings.yaml
kubectl delete -f ../wiremock/configmaps/ni-responses.yaml

# Remove namespace
kubectl delete namespace wiremock
```
