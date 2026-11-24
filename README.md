# Docker & Kubernetes Deployment for a MERN Chat App

A DevOps project that packages a three-service application into Docker containers and deploys it with Kubernetes. The chat app provides a practical workload for learning how to build images, connect services, inject configuration, and troubleshoot failed deployments.

## Deployment architecture

| Component | Container | Port | Role |
| --- | --- | --- | --- |
| Frontend | Nginx serving a React build | `80` | Serves the UI and forwards `/api/` and `/socket.io/` traffic to the backend |
| Backend | Node.js and Express | `5001` | Handles API requests and real-time connections |
| Database | MongoDB | `27017` | Stores users and messages |

The browser uses one entry point at `localhost:8080`. Nginx forwards requests inside the deployment to `backend:5001`, so local testing only requires exposing the frontend.

## Docker and Docker Compose

- **Multi-stage builds.** The frontend Dockerfile builds React with Node.js and copies the static output into Nginx. The backend Dockerfile installs production dependencies with `npm ci` and runs the API as a non-root user.
- **Container networking.** All three services join the `chat-network` bridge network. Containers reach each other by service name. For example, the backend connects to MongoDB at `mongo:27017`.
- **Persistent storage.** Compose mounts the named volume `mongodb_data` at `/data/db`, allowing MongoDB data to survive container replacement.
- **Runtime configuration.** The backend reads its settings from `.env`. The repository includes `.env.example` so each environment can supply its own credentials.
- **Startup behavior.** `depends_on` defines startup order, and `restart: unless-stopped` restarts containers after they exit. Database readiness can be checked through the backend’s `/health` endpoint.

### Run locally

From the project root in PowerShell:

```powershell
Copy-Item .env.example .env
```

Set the MongoDB credentials, `MONGODB_URI`, and `JWT_SECRET` in `.env`. For Compose, use `mongo` as the database hostname. Then run:

```powershell
docker compose up -d --build
docker compose ps
docker compose logs --tail=50 backend
```

Open **http://localhost:8080**. To check database connectivity, open **http://localhost:5001/health** and look for `"database": "connected"`.

## Kubernetes deployment

The `k8s/` directory contains the manifests for running the same workload in a cluster.

| Kubernetes resource | How this project uses it |
| --- | --- |
| **Namespace** | `chat-app` groups the application’s Deployments, Services, and Secrets. |
| **Deployments** | Separate Deployments manage the frontend, backend, and MongoDB Pods. Each starts with one replica. |
| **ClusterIP Services** | `frontend`, `backend`, and `mongodb` give the Pods stable internal addresses. The database hostname in Kubernetes is `mongodb`. |
| **Secrets** | `mongo-admin` supplies MongoDB initialization credentials, `backend-db` supplies `MONGODB_URI`, and `backend-auth` supplies `JWT_SECRET`. |
| **PersistentVolume and claim** | The manifests define a 5 GiB `hostPath` volume and a matching claim. The MongoDB container still needs a mount at `/data/db` to use that storage. |
| **Port forwarding** | `kubectl port-forward` makes the frontend accessible on the developer’s laptop while keeping the Services internal. |

### Deploy and verify

Create the namespace first:

```powershell
kubectl apply -f k8s/namespace.yml
```

Create the three Secrets in `chat-app` using your own values. The `mongo-admin` Secret requires `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`. Use the same credentials in `MONGODB_URI`, with `mongodb:27017` as the host and `authSource=admin`.

```powershell
kubectl apply -f k8s/
kubectl rollout status deployment/backend-deployment -n chat-app
kubectl get pods,svc -n chat-app
kubectl logs deployment/backend-deployment -n chat-app --tail=50
kubectl port-forward service/frontend -n chat-app 8080:80
```

Keep the port-forward terminal open and visit **http://localhost:8080**.

### Troubleshooting a restarting Pod

During setup, the backend repeatedly restarted while the frontend and MongoDB Pods stayed running. The previous container’s logs identified the cause:

```powershell
kubectl logs deployment/backend-deployment -n chat-app --previous
```

The application reported `MONGODB_URI environment variable is required`. Referencing the `backend-db` Secret in the Deployment supplies the missing value. The backend also uses `PORT=5001` to match its Service and `JWT_SECRET` to sign login sessions.

## Jenkins pipeline

The included `Jenkinsfile` checks out source code, runs HTTP checks, and deploys with Docker Compose. Kubernetes deployment is handled separately through the manifests. To use Jenkins with a new fork, update the checkout repository and branch, run the checks after deployment, and point the frontend check at Compose host port `8080`.

## Source and license

Adapted from [iemafzalhassan/full-stack_chatApp](https://github.com/iemafzalhassan/full-stack_chatApp). The original [MIT license](LICENSE) and copyright notice are retained.
