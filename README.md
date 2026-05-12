# Kubernetes Setup: PostgreSQL StatefulSet + API

This guide explains how to deploy a PostgreSQL StatefulSet and a stateless API that connects to it, exposing only the API via Ingress.

## Architecture

```
Internet → Traefik Ingress (meicm.pt:80) → API Service → API Pods (2) → postgres:5432 → postgres-0
```

## Prerequisites

- k3s cluster (running on Lima VMs)
- Docker for building images
- Docker Hub account for image registry

## Project Structure

```
cc-lab8/
├── postgres/           # PostgreSQL StatefulSet
│   ├── statefulset.yaml
│   ├── service.yaml
│   ├── secret.yaml
│   └── configmap.yaml
├── api/                # API Deployment
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── secret.yaml
│   └── configmap.yaml
└── ingress/            # Traefik Ingress
    └── ingress.yaml
```

## Step 1: Create the API Application

Create a simple FastAPI application that connects to PostgreSQL.

**app.py**
```python
from fastapi import FastAPI
import psycopg2
import os

app = FastAPI()

def get_db():
    return psycopg2.connect(
        host=os.environ.get("DB_HOST", "postgres"),
        port=int(os.environ.get("DB_PORT", "5432")),
        user=os.environ.get("DB_USER", "postgres"),
        password=os.environ.get("DB_PASSWORD", "postgres"),
        database=os.environ.get("DB_NAME", "appdb")
    )

@app.get("/")
def root():
    return {"status": "ok", "message": "API running"}

@app.get("/health")
def health():
    try:
        conn = get_db()
        conn.close()
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "database": "disconnected", "error": str(e)}

@app.get("/db")
def db_test():
    try:
        conn = get_db()
        cur = conn.cursor()
        cur.execute("SELECT version();")
        version = cur.fetchone()[0]
        conn.close()
        return {"database": "postgres", "version": version}
    except Exception as e:
        return {"error": str(e)}
```

**Dockerfile**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install fastapi uvicorn psycopg2-binary
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

## Step 2: Build and Push the API Image

```bash
# Build the image locally
docker build -t api:latest .

# Tag for Docker Hub
docker tag api:latest <your-dockerhub-username>/api:latest

# Login and push to Docker Hub
docker login
docker push <your-dockerhub-username>/api:latest
```

## Step 3: Create ConfigMaps and Secrets

Create separate files for configuration to keep sensitive data secure.

**postgres/secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
```

**postgres/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: appdb
```

**api/secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
type: Opaque
stringData:
  DB_USER: postgres
  DB_PASSWORD: postgres
```

**api/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: appdb
```

## Step 4: Create PostgreSQL StatefulSet

**postgres/statefulset.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

**postgres/service.yaml** (Headless service for StatefulSet DNS)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
  clusterIP: None
```

## Step 5: Create API Deployment

**api/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: <your-dockerhub-username>/api:latest
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: api-secret
            - configMapRef:
                name: api-config
```

**api/service.yaml** (ClusterIP - not exposed directly)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Step 6: Create Ingress

**ingress/ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
spec:
  ingressClassName: traefik
  rules:
    - host: meicm.pt
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

## Step 7: Deploy to Cluster

```bash
# Apply all resources
kubectl apply -f postgres/
kubectl apply -f api/
kubectl apply -f ingress/

# Verify deployment
kubectl get all,ingress

# Watch pods
kubectl get pods -w
```

## Step 8: Test

```bash
curl http://meicm.pt/
curl http://meicm.pt/health
curl http://meicm.pt/db
```

## Troubleshooting

**Image pull errors**
- Ensure image is pushed to Docker Hub: `docker push <username>/api:latest`
- Check pods: `kubectl get pods`
- Describe pod: `kubectl describe pod <pod-name>`

**Ingress not working**
- Check ingress status: `kubectl get ingress`
- Check Traefik service: `kubectl get svc -A | grep traefik`
- Test with curl using Host header: `curl -H "Host: meicm.pt" http://<traefik-ip>:31130/`

**Database connection issues**
- Check postgres is running: `kubectl get pods -l app=postgres`
- Check DNS resolution: `kubectl exec -it api-<pod> -- nslookup postgres`
- Check postgres logs: `kubectl logs postgres-0`

## Cleanup

```bash
kubectl delete -f postgres/ api/ ingress/
```

Or delete everything at once:
```bash
kubectl delete all --all
```