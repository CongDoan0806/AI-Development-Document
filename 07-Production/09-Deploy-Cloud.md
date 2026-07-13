# Deploy To Cloud

## 1. Deployment Options

| Platform | Best For |
|----------|----------|
| **Vercel/Netlify** | Frontend + serverless functions |
| **AWS Lambda** | Serverless API |
| **AWS ECS/Fargate** | Containers |
| **AWS EKS / GKE** | Kubernetes |
| **Render / Railway** | Quick deploy |
| **Modal / Replicate** | ML workloads |
| **HuggingFace Spaces** | Demos |
| **LangGraph Platform** | Native LangGraph hosting |

---

## 2. Docker (Foundation)

### 2.1 Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install deps first (for caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY . .

# Non-root user
RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["gunicorn", "main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

### 2.2 .dockerignore

```
__pycache__
*.pyc
.git
.env
venv/
*.db
chroma_db/
```

### 2.3 Multi-Stage Build

```dockerfile
FROM python:3.11 AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH

CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

---

## 3. Docker Compose (Local Dev)

```yaml
version: "3.8"

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
    depends_on:
      - redis
      - postgres
    volumes:
      - ./chroma_db:/app/chroma_db
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
  
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=db
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api

volumes:
  redis_data:
  postgres_data:
```

```bash
docker-compose up -d
docker-compose logs -f api
docker-compose scale api=3  # Scale
```

---

## 4. AWS ECS / Fargate

### 4.1 ECR Push

```bash
aws ecr create-repository --repository-name my-ai-app

# Login
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI

# Build & push
docker build -t my-ai-app .
docker tag my-ai-app:latest $ECR_URI/my-ai-app:latest
docker push $ECR_URI/my-ai-app:latest
```

### 4.2 Task Definition

```json
{
  "family": "ai-app",
  "containerDefinitions": [{
    "name": "api",
    "image": "$ECR_URI/my-ai-app:latest",
    "memory": 512,
    "cpu": 256,
    "portMappings": [{"containerPort": 8000}],
    "environment": [
      {"name": "OPENAI_API_KEY", "value": "..."}
    ],
    "secrets": [
      {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:..."}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/ai-app",
        "awslogs-region": "us-east-1"
      }
    }
  }]
}
```

### 4.3 Service With ALB

```bash
aws ecs create-service \
    --cluster prod-cluster \
    --service-name ai-app \
    --task-definition ai-app:1 \
    --desired-count 3 \
    --load-balancers targetGroupArn=$TG_ARN,containerName=api,containerPort=8000
```

---

## 5. Kubernetes (EKS/GKE)

### 5.1 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ai-app
  template:
    metadata:
      labels:
        app: ai-app
    spec:
      containers:
      - name: api
        image: my-ai-app:latest
        ports:
        - containerPort: 8000
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: openai-secret
              key: api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: ai-app
spec:
  selector:
    app: ai-app
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

### 5.2 HPA Auto-Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 5.3 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: ai-app-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ai-app
            port:
              number: 80
```

---

## 6. Serverless

### 6.1 AWS Lambda

```python
# lambda_handler.py
from mangum import Mangum  # FastAPI → Lambda adapter
from main import app

handler = Mangum(app)
```

`serverless.yml`:
```yaml
service: ai-app

provider:
  name: aws
  runtime: python3.11
  region: us-east-1
  environment:
    OPENAI_API_KEY: ${env:OPENAI_API_KEY}

functions:
  api:
    handler: lambda_handler.handler
    timeout: 30
    memorySize: 1024
    events:
      - httpApi: "*"
```

⚠️ **Lambda limitations**:
- 15 min timeout
- Cold start ~1-3s
- Cannot persist state
- Vector DB phải external

### 6.2 Modal (ML-Focused)

```python
import modal

stub = modal.Stub("ai-app")
image = modal.Image.debian_slim().pip_install("langchain", "langchain-openai")

@stub.function(image=image, secret=modal.Secret.from_name("openai"))
@modal.web_endpoint(method="POST")
def query(request: dict):
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI()
    return llm.invoke(request["question"]).content

# Deploy
# modal deploy main.py
```

---

## 7. Vercel (Serverless Functions)

```python
# api/chat.py
from langchain_openai import ChatOpenAI

llm = ChatOpenAI()

def handler(request, response):
    body = request.json()
    result = llm.invoke(body["question"])
    return response.json({"answer": result.content})
```

```json
// vercel.json
{
  "functions": {
    "api/chat.py": {
      "maxDuration": 30,
      "memory": 1024
    }
  }
}
```

---

## 8. LangGraph Cloud / Platform

Specifically cho LangGraph apps:

```bash
# Install CLI
pip install langgraph-cli

# Test locally
langgraph dev

# Deploy
langgraph deploy --config langgraph.json
```

`langgraph.json`:
```json
{
  "graphs": {
    "my-agent": "./main.py:graph"
  },
  "dependencies": ["langchain", "langgraph"],
  "env": ".env"
}
```

→ Auto: scaling, persistence (checkpoints), monitoring.

---

## 9. CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install -r requirements.txt
      - run: pytest
  
  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/login-action@v2
        with:
          registry: ${{ secrets.ECR_URI }}
          username: ${{ secrets.AWS_ACCESS_KEY_ID }}
          password: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ secrets.ECR_URI }}/ai-app:${{ github.sha }}
  
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster prod \
            --service ai-app \
            --force-new-deployment
```

---

## 10. Environment Management

### dev / staging / prod

```bash
# .env.dev
OPENAI_API_KEY=sk-dev-...
LANGCHAIN_PROJECT=ai-app-dev

# .env.prod
OPENAI_API_KEY=sk-prod-...
LANGCHAIN_PROJECT=ai-app-prod
```

```python
import os
from pathlib import Path
from dotenv import load_dotenv

env = os.environ.get("ENVIRONMENT", "dev")
env_file = Path(f".env.{env}")
if env_file.exists():
    load_dotenv(env_file)
```

---

## 11. Blue-Green Deployment

```
Blue (current production) ──→ Users
Green (new version)       ──→ Staging traffic

After validation: swap Blue ↔ Green
```

### Kubernetes Implementation

```yaml
# Service points to green
apiVersion: v1
kind: Service
metadata:
  name: ai-app
spec:
  selector:
    app: ai-app
    version: green  # Switch this
```

---

## 12. Canary Deployment

```
90% → Blue (v1)
10% → Green (v2)

If v2 OK → 50/50 → 100% v2
If v2 bad → rollback to 100% v1
```

Istio/Linkerd có thể split traffic.

---

## 13. Health Checks & Readiness

```python
@app.get("/health")
async def health():
    return {"status": "ok"}

@app.get("/ready")
async def ready():
    """Detailed check"""
    checks = {
        "llm": False,
        "vector_db": False,
        "cache": False,
    }
    
    try:
        await llm.ainvoke("ping")
        checks["llm"] = True
    except: pass
    
    try:
        vectorstore.similarity_search("test", k=1)
        checks["vector_db"] = True
    except: pass
    
    try:
        await redis.ping()
        checks["cache"] = True
    except: pass
    
    ready = all(checks.values())
    status_code = 200 if ready else 503
    return JSONResponse(checks, status_code=status_code)
```

---

## 14. Cost Comparison

```
1M requests/month, 1KB avg payload:

AWS Lambda:        $200 (compute) + $10 (egress) = $210
AWS Fargate:       $150 + $5 = $155
AWS EKS:           $73 (control plane) + EC2 cost
GCP Cloud Run:     $180
Vercel:            $200 (Pro plan)
Modal:             $400 (premium)
Self-host VPS:     $50-200
```

→ Lambda OK cho low scale. ECS/k8s tốt cho consistent traffic.

---

## 15. Best Practices

✅ **Container everything**: portable, reproducible

✅ **Infrastructure as Code**: Terraform, Pulumi

✅ **Multi-region**: failover, low latency

✅ **Automated rollback**: nếu error rate spike

✅ **Cost monitoring**: alert nếu cost > budget

✅ **Secret rotation**: tự động rotate API keys

✅ **Backup**: vector DB, configs

---

## 16. Demo: Full Deploy

```bash
# 1. Build
docker build -t my-ai-app .

# 2. Test locally
docker run -p 8000:8000 --env-file .env my-ai-app

# 3. Push
docker tag my-ai-app:latest $ECR_URI/my-ai-app:v1.0
docker push $ECR_URI/my-ai-app:v1.0

# 4. Deploy
aws ecs update-service --cluster prod --service ai-app --force-new-deployment

# 5. Verify
curl https://api.example.com/health
```

---

## 17. Checklist Phase 7

- [ ] Dockerfile production-ready
- [ ] Docker Compose for local
- [ ] Cloud deploy (ECS/EKS/Lambda/Vercel)
- [ ] CI/CD pipeline
- [ ] Environment management
- [ ] Health checks
- [ ] Auto-scaling
- [ ] Monitoring & alerts
- [ ] Cost tracking

🎉 **Hoàn thành Phase 7 - Production!**

➡️ **Tiếp theo**: [Phase 8 - Real Projects](../08-Real-Projects/01-Chatbot-PDF.md)
