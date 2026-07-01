# SkyCheck — Cloud-Native Weather App

A two-service microservices weather application deployed on Kubernetes. The frontend is a static HTML/JS app served by nginx, and the backend is a Node.js Express API that fetches real-time weather data from Open-Meteo (no API key required).

---

## Architecture

```
User → [Frontend: nginx on port 80]
             ↓ HTTP fetch
       [Backend API: Node.js on port 3001]
             ↓ HTTP fetch
       [Open-Meteo Public API]
```

The frontend and backend run as separate Kubernetes Deployments, each with their own Service. The frontend is exposed externally via NodePort, while the backend is internal-only via ClusterIP — only the frontend can reach it within the cluster.

---

## Services

| Service          | Tech          | Port | K8s Service Type |
|------------------|---------------|------|------------------|
| weather-frontend | nginx (HTML)  | 80   | NodePort         |
| weather-api      | Node.js/Express | 3001 | ClusterIP      |

---

## Key Design Decisions

**Why two separate services instead of one?**
Separating the frontend and backend follows the single responsibility principle — the frontend handles presentation, the backend handles data fetching and business logic. This lets each service scale, deploy, and update independently.

**Why ClusterIP for the backend?**
The backend API has no reason to be publicly accessible. Using ClusterIP keeps it internal to the cluster so only the frontend service can reach it, reducing the attack surface.

**Why Open-Meteo?**
Open-Meteo is a free, open-source weather API that requires no API key. This eliminates secrets management complexity for a demo project and makes the app immediately runnable without any configuration.

**Why nginx for the frontend?**
A static HTML file doesn't need a Node.js server. nginx is lightweight, fast, and purpose-built for serving static files. It keeps the frontend image small and the container resource usage minimal.

**Why GHCR over Docker Hub?**
GitHub Container Registry integrates natively with GitHub Actions using `GITHUB_TOKEN` — no extra secrets needed. It keeps the entire workflow (code, CI, registry) in one place.

---

## CI/CD Pipeline

On every push to `main`, GitHub Actions automatically:
1. Checks out the code
2. Logs into GHCR using `GITHUB_TOKEN` (no manual secrets needed)
3. Builds and pushes the backend image to GHCR
4. Builds and pushes the frontend image to GHCR

Both images are tagged with `latest` and the commit SHA for traceability.

---

## Running Locally

### Prerequisites
- Docker
- minikube
- kubectl

### Steps

**1. Start minikube**
```bash
minikube start
```

**2. Update image names**

In `k8s/backend-deployment.yaml` and `k8s/frontend-deployment.yaml`, replace `YOUR_GITHUB_USERNAME` with your actual GitHub username.

**3. Apply Kubernetes manifests**
```bash
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
```

**4. Verify pods are running**
```bash
kubectl get pods
```

**5. Get the frontend URL**
```bash
minikube service weather-frontend --url
```

Open that URL in your browser and search for any city.

**6. (Optional) Test the backend directly**
```bash
kubectl port-forward svc/weather-api 3001:3001
curl "http://localhost:3001/weather?city=Minneapolis"
```

---

## Running with Docker Compose (local dev)

```bash
docker compose up --build
```

Then open `http://localhost:8080` in your browser.

---

## Project Structure

```
weather-app/
├── backend/
│   ├── server.js          # Express API
│   ├── package.json
│   └── Dockerfile
├── frontend/
│   ├── index.html         # Single-page weather UI
│   ├── nginx.conf         # nginx static server config
│   └── Dockerfile
├── k8s/
│   ├── backend-deployment.yaml
│   └── frontend-deployment.yaml
├── .github/
│   └── workflows/
│       └── build-push.yml  # CI/CD pipeline
└── README.md
```

---

## Observability

Both services include health check endpoints used by Kubernetes liveness and readiness probes:

- `GET /health` — backend returns `{ status: "ok", service: "weather-api" }`
- `GET /health` — frontend nginx returns `200 ok`

Resource requests and limits are defined in both deployment manifests to prevent resource contention in the cluster.
