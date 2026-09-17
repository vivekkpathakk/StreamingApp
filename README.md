# StreamingApp

Stream premium video content, host live watch parties, and manage your catalogue with a modern microservice architecture. The platform now ships with a production-ready admin portal, real-time chat, S3-backed adaptive streaming, and a redesigned cinematic frontend experience.

## Architecture

| Service | Port | Description |
| --- | --- | --- |
| `authService` | 3001 | User authentication, registration, JWT issuance |
| `streamingService` | 3002 | Video catalogue, S3 playback endpoints, public APIs |
| `adminService` | 3003 | Dedicated admin microservice for asset management and uploads |
| `chatService` | 3004 | Websocket + REST chat for live watch parties |
| `frontend` | 3000 | React SPA with revamped UI and integrated chat |
| `mongo` | 27017 | Shared MongoDB instance |

All backend services share common database models and utilities through `backend/common`.

## Environment Configuration

Create an `.env` for each service (or export variables before running). All services accept the standard AWS credentials for S3 access.

### Auth Service (`backend/authService/.env`)
```ini
PORT=3001
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Streaming Service (`backend/streamingService/.env`)
```ini
PORT=3002
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=
STREAMING_PUBLIC_URL=http://localhost:3002
```

### Admin Service (`backend/adminService/.env`)
```ini
PORT=3003
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Chat Service (`backend/chatService/.env`)
```ini
PORT=3004
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
```

### Frontend build variables (`frontend/.env` or Docker build args)
```ini
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

## Running with Docker Compose

1. Populate the environment variables above (or rely on the defaults baked into `docker-compose.yml`).
2. Build and start the stack:
   ```bash
   docker-compose up --build
   ```
3. Navigate to `http://localhost:3000` for the web app.

The compose file provisions MongoDB plus all four Node.js microservices. S3 credentials are optional for local testing—you can still browse seeded metadata, but streaming requires valid S3 objects.

## Local Development

Install dependencies for each service:

```bash
# auth service
cd backend/authService && npm install

# streaming service
cd ../streamingService && npm install

# admin service
cd ../adminService && npm install

# chat service
cd ../chatService && npm install

# frontend
cd ../../frontend && npm install
```

Run the services (in separate terminals) after starting MongoDB:

```bash
cd backend/authService && npm run dev
cd backend/streamingService && npm run dev
cd backend/adminService && npm run dev
cd backend/chatService && npm run dev
cd frontend && npm start
```

## Feature Highlights

- **S3-backed adaptive streaming** with secure signed uploads for admins.
- **Dedicated admin microservice** for video ingestion, metadata management, and featured curation.
- **Real-time chat** overlay in the player (Socket.IO + persistent message history).
- **Modern React experience** featuring cinematic hero sections, dynamic carousels, and responsive design.
- **Role-aware access control** across frontend routes and backend microservices.

## Testing

Automated tests are not yet included. Recommended smoke checks:

1. Register and log in through the web UI.
2. Upload a small video + thumbnail via the admin dashboard (requires valid S3 credentials).
3. Confirm playback from the browse page and verify that chat messages broadcast between multiple browser tabs.

## License

MIT © StreamFlix Team

---

# DevOps Infrastructure & EKS Deployment Specs

Automated DevOps pipeline deploying a high-availability MERN streaming application to Amazon EKS using Jenkins CI/CD, AWS ECR, and CloudWatch Container Insights.

## Architecture & System Flow

```
+-----------------------------------------------------------------------------------+
|                                 DEVELOPMENT PIPELINE                              |
+-----------------------------------------------------------------------------------+
|  [ Developer ] ---> Git Push ---> [ GitHub Repo ]                                 |
|                                         |                                         |
|                                         v                                         |
|                             [ Jenkins CI/CD Pipeline ]                            |
|                                         |                                         |
|          +------------------------------+------------------------------+          |
|          |                              |                              |          |
|          v                              v                              v          |
|  ( ECR Login )               ( Build & Push Images )        ( Deploy Manifests )  |
|                                         |                              |          |
+-----------------------------------------|------------------------------|----------+
                                          |                              |
+-----------------------------------------|------------------------------|----------+
|                                    AWS CLOUD                           v          |
+-----------------------------------------|-----------------------------------------+
|                                         v                                         |
|                            [ Amazon ECR Repositories ]                            |
|                                                                                   |
|  [ Amazon EKS Cluster: streaming-cluster-v3 ]                                     |
|  +-----------------------------------------------------------------------------+  |
|  |  +-------------------------+      +--------------------------------------+  |  |
|  |  |   AWS Elastic LoadBal   | ---> | Frontend Service (Nginx / React Port)|  |  |
|  |  +-------------------------+      +--------------------------------------+  |  |
|  |                                                   |                         |  |
|  |       +-------------------+-----------------------+-------------------+     |  |
|  |       |                   |                       |                   |     |  |
|  |       v                   v                       v                   v     |  |
|  |  (Auth: 3001)       (Admin: 3003)           (Chat: 3004)     (Streaming: 3002)|  |
|  |       |                   |                       |                   |     |  |
|  |       +-------------------+-----------+-----------+-------------------+     |  |
|  |                                       |                                     |  |
|  |                                       v                                     |  |
|  |                            (MongoDB Service: 27017)                         |  |
|  +-----------------------------------------------------------------------------+  |
|                                          |                                        |
|                                          v                                        |
|                 [ AWS CloudWatch Container Insights & Logs ]                      |
+-----------------------------------------------------------------------------------+
```

### 1. Development & CI/CD Pipeline
- **Developer Push**: Code updates are committed and pushed to the GitHub repository (`main` branch).
- **Jenkins Orchestration**: Jenkins automatically pulls source code and triggers the pipeline workflow.
- **ECR Authentication**: Jenkins logs into AWS ECR using system credentials (`aws-creds-v3`).
- **Build & Push**: Docker images for Frontend, Auth, Admin, Chat, and Streaming services are compiled, tagged, and pushed to individual AWS ECR repositories.
- **EKS Deployment**: Jenkins configures cluster `kubeconfig` and executes `kubectl apply -f app-deployment.yaml` for zero-downtime rolling updates.

### 2. AWS EKS Infrastructure (`streaming-cluster-v3`)
- **Public Ingress Layer**: AWS Elastic Load Balancer (ELB) receives public HTTP traffic and forwards requests to the `frontend-service` (React running on Nginx, Port 80).
- **Microservices Routing Layer**: Internal requests route to isolated Node.js services via Kubernetes ClusterIP services:
  - **Auth Service**: Port `3001` (`auth-service`)
  - **Streaming Service**: Port `3002` (`streaming-service`)
  - **Admin Service**: Port `3003` (`admin-service`)
  - **Chat Service**: Port `3004` (`chat-service`)
- **Database Layer**: All microservices persist and retrieve data through a dedicated MongoDB service (`mongo`) bound to Port `27017`.

### 3. Observability & Telemetry
- **CloudWatch Control Plane Logging**: EKS control plane audit, API, and authenticator logs are pushed to `/aws/eks/streaming-cluster-v3/cluster`.
- **Container Insights**: The `amazon-cloudwatch-observability` addon (Fluent Bit daemonset & CloudWatch agent) collects pod telemetry and forwards application logs to AWS CloudWatch.

## Technical & Project Specifications
- **Upstream Repository**: `https://github.com/UnpredictablePrashant/StreamingApp.git`
- **AWS Account ID**: `138893339858` (`us-east-1`)
- **EKS Cluster**: `streaming-cluster-v3`
- **Container Registries (ECR)**: `streaming-frontend`, `streaming-auth`, `streaming-admin`, `streaming-chat`, `streaming-service`
- **Jenkins Credentials ID**: `aws-creds-v3`
- **Live Application Endpoint**: `http://a0a1f7fce9c14bfc89893bc35141a88-1356062452.us-east-1.elb.amazonaws.com`

## Component Port & Service Binding
| Component | Type | Internal Port | Service Type | EKS Service Name |
| :--- | :--- | :--- | :--- | :--- |
| **Database** | MongoDB 6 | `27017` | ClusterIP | `mongo` |
| **Auth Microservice** | Node.js | `3001` | ClusterIP | `auth-service` |
| **Admin Microservice** | Node.js | `3003` | ClusterIP | `admin-service` |
| **Chat Microservice** | Node.js | `3004` | ClusterIP | `chat-service` |
| **Streaming Microservice** | Node.js | `3002` | ClusterIP | `streaming-service` |
| **Frontend Application** | React / Nginx | `80` | LoadBalancer | `frontend-service` |

## CI/CD Pipeline Workflow (`Jenkinsfile`)
1. **Declarative Checkout**: Clones latest commits from `origin/main`.
2. **ECR Login**: Authenticates Docker daemon to AWS ECR in `us-east-1` using global `aws-creds-v3` credentials.
3. **Build & Push Frontend**: Compiles production React static files into Nginx base image and pushes `streaming-frontend:latest` to ECR.
4. **Build & Push Microservices**: Builds isolated Docker contexts for Auth, Admin, Chat, and Streaming services and pushes tags to ECR.
5. **Deploy to EKS**: Updates `kubeconfig` context for `streaming-cluster-v3` and applies `app-deployment.yaml`.

## Operational Verification Commands

**1. Check running pods across all namespaces:**

```
$ kubectl get pods -A

NAMESPACE           NAME                                                  READY   STATUS    RESTARTS   AGE
amazon-cloudwatch    amazon-cloudwatch-observability-controller-manager-6887c87g4l2l   1/1   Running   0   12d
amazon-cloudwatch    cloudwatch-agent-lr4df                               1/1   Running   0   12d
amazon-cloudwatch    cloudwatch-agent-mkqmg                               1/1   Running   0   12d
amazon-cloudwatch    fluent-bit-s58r5                                     1/1   Running   0   12d
amazon-cloudwatch    fluent-bit-wrcfw                                     1/1   Running   0   12d
default              admin-deployment-cbdbccff6-466dn                     1/1   Running   1 (12d ago)   12d
default              auth-deployment-854ff764c5-gkhd8                     1/1   Running   0   12d
default              chat-deployment-58dbb67b87-qn69j                     1/1   Running   0   12d
default              frontend-deployment-cc65dd479-2gdvz                  1/1   Running   0   12d
default              frontend-deployment-cc65dd479-bmw7l                  1/1   Running   0   12d
default              mongo-deployment-7c94fcd666-wvk68                    1/1   Running   0   12d
default              streaming-deployment-b9f8559d4-dgwfm                 1/1   Running   0   12d
kube-system          aws-node-jnsvt                                       2/2   Running   0   14d
kube-system          aws-node-mgv8k                                       2/2   Running   0   14d
kube-system          coredns-7b9bfc5446-kbqk2                             1/1   Running   0   17d
kube-system          coredns-7b9bfc5446-kwl4m                             1/1   Running   0   17d
kube-system          kube-proxy-g89ks                                     1/1   Running   0   14d
kube-system          kube-proxy-kwl8p                                     1/1   Running   0   14d
```

**2. Check public service load balancer endpoint:**

```
$ kubectl get svc frontend-service

NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
frontend-service   LoadBalancer   10.100.100.144   a0a1f7fce9c14bfc89893bc35141a88-1356062452.us-east-1.elb.amazonaws.com   80:30342/TCP   12d
```

**3. Inspect CloudWatch logging agent status:**

```
$ kubectl get pods -n amazon-cloudwatch

NAME                                                              READY   STATUS    RESTARTS   AGE
amazon-cloudwatch-observability-controller-manager-6887c87g4l2l   1/1     Running   0          12d
cloudwatch-agent-lr4df                                            1/1     Running   0          12d
cloudwatch-agent-mkqmg                                            1/1     Running   0          12d
fluent-bit-s58r5                                                  1/1     Running   0          12d
fluent-bit-wrcfw                                                  1/1     Running   0          12d
```
