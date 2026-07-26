# 14 — Deployment: Docker, Kubernetes, Azure

**Prerequisites:** A working app (Chapters 01–10 at minimum).

## 14.1 Docker

`samples/Demo/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish samples/Demo/MyNLWeb.Demo.csproj -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyNLWeb.Demo.dll"]
```

`.dockerignore` (don't ship secrets or build artifacts into the image):

```
**/bin
**/obj
**/.vs
**/secrets.json
.git
```

`docker-compose.yml` for local multi-container dev (API + vector DB, mirroring Chapter 13's topology without full Aspire):

```yaml
services:
  api:
    build:
      context: .
      dockerfile: samples/Demo/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - AzureOpenAI__ApiKey=${AZURE_OPENAI_API_KEY}
      - AzureOpenAI__Endpoint=${AZURE_OPENAI_ENDPOINT}
    depends_on:
      - qdrant

  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
```

**Never bake API keys into the image** — pass them at runtime via `environment:`/`docker run -e`, sourced from your host's env or a `.env` file that's gitignored.

```bash
docker-compose up --build
```

## 14.2 Kubernetes

A minimal Deployment + Service. Secrets go in a Kubernetes `Secret`, never inlined:

```bash
kubectl create secret generic mynlweb-secrets \
  --from-literal=azure-openai-api-key=YOUR_KEY \
  --from-literal=azure-openai-endpoint=YOUR_ENDPOINT
```

`deployment/kubernetes/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynlweb-api
spec:
  replicas: 2
  selector:
    matchLabels: { app: mynlweb-api }
  template:
    metadata:
      labels: { app: mynlweb-api }
    spec:
      containers:
        - name: api
          image: your-registry/mynlweb-api:latest
          ports: [{ containerPort: 8080 }]
          env:
            - name: AzureOpenAI__ApiKey
              valueFrom: { secretKeyRef: { name: mynlweb-secrets, key: azure-openai-api-key } }
            - name: AzureOpenAI__Endpoint
              valueFrom: { secretKeyRef: { name: mynlweb-secrets, key: azure-openai-endpoint } }
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 5
          livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: mynlweb-api
spec:
  selector: { app: mynlweb-api }
  ports: [{ port: 80, targetPort: 8080 }]
  type: ClusterIP
```

The `/health` endpoint from Chapter 10.4 is exactly what the readiness/liveness probes hit — build health checks *before* you write Kubernetes manifests, not after, or you'll be tempted to point probes at `/` and get false-positive "healthy" pods whose backend or AI provider is actually down.

```bash
kubectl apply -f deployment/kubernetes/deployment.yaml
```

Multiple `replicas` works cleanly here because the pipeline is stateless per-request — no in-memory session state to worry about, as long as your `IDataBackend`/vector DB is itself a shared external service, not in-process (which is why `MockDataBackend`'s in-memory list is fine for one dev box but wouldn't be for a multi-replica deployment).

## 14.3 Azure

Two common targets: **Azure Container Apps** (serverless-ish, scales to zero, simplest) or **Azure App Service** (more traditional PaaS). A basic Container Apps deploy via Azure CLI:

```bash
az group create -n mynlweb-rg -l eastus

az containerapp env create -n mynlweb-env -g mynlweb-rg -l eastus

az containerapp create \
  -n mynlweb-api -g mynlweb-rg --environment mynlweb-env \
  --image your-registry/mynlweb-api:latest \
  --target-port 8080 --ingress external \
  --secrets azure-openai-key=YOUR_KEY \
  --env-vars AzureOpenAI__ApiKey=secretref:azure-openai-key AzureOpenAI__Endpoint=YOUR_ENDPOINT
```

For Azure OpenAI specifically, prefer **Managed Identity + `DefaultAzureCredential`** (shown in Chapter 05.4) over API keys in production — it removes a secret from the deployment entirely:

```bash
az containerapp identity assign -n mynlweb-api -g mynlweb-rg --system-assigned
# then grant that identity the "Cognitive Services OpenAI User" role on your Azure OpenAI resource
```

## 14.4 Pre-deployment checklist

Before your first real deployment, confirm:

- [ ] No secrets in `appsettings.json` or the Docker image layers (check with `docker history`)
- [ ] Rate limiting is on (Chapter 10.2) — an exposed `/ask` endpoint without it is an open LLM-cost spigot
- [ ] Health checks wired to actual readiness/liveness probes, not just `/`
- [ ] `/mcp` endpoint access is scoped the same as `/ask` — don't accidentally leave one authenticated and the other open
- [ ] Structured logging includes `QueryId` so you can trace a user-reported bad answer back through logs
- [ ] Authentication decision made deliberately, not by omission — the reference implementation ships **no built-in auth** (`doc/design-decisions.md` §1); decide whether you're adding API-key middleware, OAuth, or a gateway-level auth layer *before* going live, not after

Next: [`15-build-order-checklist.md`](15-build-order-checklist.md) — the end-to-end sequence tying every chapter together.
