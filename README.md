# MERN Chat | Docker, Kubernetes & Jenkins

A real-time chat application packaged for two deployment workflows. Docker Compose runs the frontend, API and MongoDB together on one host. Kubernetes manifests describe the same components as Deployments and Services in the `chat-app` namespace.

## How a message travels

React sends requests to the same origin that served the page. Nginx serves the frontend and forwards `/api/` and `/socket.io/` to Express on port 5001. Express stores users and messages in MongoDB. Socket.IO delivers live updates and online status. Signup and login use a JWT in an HTTP-only cookie.

## Run with Docker Compose

From the project root, with Docker Desktop running:

1. Copy the configuration template with `Copy-Item .env.example .env` in PowerShell.
2. Set a unique MongoDB password and a long random JWT secret. Set `MONGODB_URI` to `mongodb://chatadmin:<encoded-password>@mongo:27017/chatapp?authSource=admin`. Use the same password in both settings, URL-encoding special characters in the URI. Cloudinary credentials enable image uploads.
3. Run `docker compose up -d --build` and open [localhost:8080](http://localhost:8080).
4. Check `docker compose ps` and [the API health endpoint](http://localhost:5001/health). Look for `database: connected`. Run `docker compose down` to stop the stack and retain the named database volume.

## Kubernetes

The manifests use prebuilt images. For a fresh local cluster, apply `k8s/namespace.yml` first. Create these Secrets in the same namespace using your own values.

| Secret | Required keys | Purpose |
| --- | --- | --- |
| mongo-admin | MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD | Initialize MongoDB |
| backend-db | MONGODB_URI | Connect to the database using the Service hostname `mongodb` and `authSource=admin` |
| backend-auth | JWT_SECRET | Sign login sessions |

Apply the remaining manifests and access the app through the frontend:

```powershell
kubectl apply -f k8s/
kubectl get pods,svc -n chat-app
kubectl logs -n chat-app deployment/backend-deployment --tail=30
kubectl port-forward -n chat-app service/frontend 8080:80
```

Keep the last terminal open while using localhost:8080. The supplied MongoDB manifest declares a 5 GiB claim but omits its container mount at `/data/db`. Plan a data migration before changing storage on an existing deployment.

## Jenkins

The supplied Jenkinsfile checks out the upstream master branch, tests HTTP endpoints, then deploys with Docker Compose. Before using it for this repository, point checkout at your own repository and branch, provide the local .env securely, and move endpoint checks after startup. The frontend check should use host port 8080. Kubernetes deployments are applied separately with kubectl.

## Source and license

Based on [iemafzalhassan/full-stack_chatApp](https://github.com/iemafzalhassan/full-stack_chatApp). The original [MIT license](LICENSE) and copyright notice are retained.
