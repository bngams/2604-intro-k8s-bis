# Pet Clinic - Kustomize Setup

This directory contains Kubernetes manifests organized with Kustomize for the Pet Clinic application.

## Structure

```
04-kustomize/
├── base/                    # Base manifests (common across all environments)
│   ├── namespaces.yml
│   ├── secret.yml
│   ├── mysql/
│   │   ├── configmap.yml
│   │   ├── statefulset.yml
│   │   ├── service.yml
│   │   └── kustomization.yml
│   ├── java-app/
│   │   ├── configmap.yml
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   ├── ingress.yml
│   │   ├── hpa.yml
│   │   └── kustomization.yml
│   └── kustomization.yml
└── overlays/                # Environment-specific configurations
    ├── local/               # Local development environment
    │   ├── kustomization.yml
    │   ├── replica-patch.yml
    │   └── ingress-patch.yml
    └── demo/                # Demo environment
        ├── kustomization.yml
        ├── replica-patch.yml
        ├── ingress-patch.yml
        └── hpa-patch.yml
```

## Usage

### Preview manifests (dry-run)

**Local environment:**
```bash
kubectl kustomize overlays/local
```

**Demo environment:**
```bash
kubectl kustomize overlays/demo
```

### Apply to cluster

**Local environment:**
```bash
kubectl apply -k overlays/local
```

**Demo environment:**
```bash
kubectl apply -k overlays/demo
```

### Delete from cluster

**Local environment:**
```bash
kubectl delete -k overlays/local
```

**Demo environment:**
```bash
kubectl delete -k overlays/demo
```

## Environment Differences

### Local Environment
- **Replicas**: 1 replica for java-app, 1 for MySQL
- **Ingress host**: `pet-clinic.local`
- **Logging**: DEBUG level
- **HPA**: 1-3 replicas, 50% CPU target
- **Labels**: `environment: local`
- **Name prefix**: `local-`

### Demo Environment
- **Replicas**: 2 replicas for java-app, 1 for MySQL
- **Ingress host**: `pet-clinic-demo.local`
- **Logging**: INFO level
- **HPA**: 2-5 replicas, 70% CPU target
- **Labels**: `environment: demo`
- **Name prefix**: `demo-`

## Testing

After applying the manifests:

1. Add the host to your `/etc/hosts`:
   ```bash
   # For local
   echo "127.0.0.1 pet-clinic.local" | sudo tee -a /etc/hosts

   # For demo
   echo "127.0.0.1 pet-clinic-demo.local" | sudo tee -a /etc/hosts
   ```

2. Access the application:
   - Local: http://pet-clinic.local
   - Demo: http://pet-clinic-demo.local

## Benefits of Kustomize

1. **DRY Principle**: Base manifests are defined once and reused
2. **Environment-specific configs**: Overlays customize for each environment
3. **No templating**: Pure YAML, no placeholders
4. **Built into kubectl**: No additional tools needed (kubectl >= 1.14)
5. **Patches**: Strategic merge and JSON patches for customization
6. **ConfigMap/Secret generation**: Built-in generators with hash suffixes
