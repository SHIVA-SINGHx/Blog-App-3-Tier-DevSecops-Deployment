# Jerney — DevSecOps-Enabled Three-Tier Blog Platform

Jerney is a full-stack blog application and DevOps/DevSecOps deployment project. It combines a React frontend, a Node.js/Express REST API, and PostgreSQL with Docker-based packaging, AWS infrastructure provisioned by Terraform, Kubernetes orchestration, and a GitHub Actions CI/CD pipeline.

The infrastructure and deployment workflow are designed to demonstrate a production-style cloud deployment. The checked-in configuration is currently a learning/demo setup and should not be treated as production-ready without the hardening items listed below.

## Features

- Browse posts, view post details, and see comments
- Create, edit, and delete posts
- Add and delete comments
- Single-page UI built with React
- REST API with a health-check endpoint and PostgreSQL persistence
- Docker Compose setup for running the complete application locally
- Terraform configuration for an AWS VPC and Amazon EKS Auto Mode cluster
- Kubernetes manifests for application services, storage, health probes, and network policies
- GitHub Actions workflow for linting, dependency checks, container builds, and infrastructure/image scanning

## Architecture

```text
Browser
  │
  ▼
React app served by Nginx (port 80 / container port 8080)
  │  /api requests
  ▼
Express REST API (port 5000)
  │
  ▼
PostgreSQL (port 5432)
```

In Kubernetes, the frontend is exposed through a NodePort service, the API is an internal ClusterIP service, and PostgreSQL uses a persistent volume claim backed by encrypted EBS storage. Network policies limit database ingress to the backend and backend ingress to the frontend. The frontend is accessed with `kubectl port-forward`; the current manifest does not configure a public Ingress or LoadBalancer.

## Screenshots

Add your screenshots to a `screenshots/` folder in the repository and use these example paths (replace the filenames with your own if needed):

### Application

![Jerney blog application home page](screenshots/app-home.png)

### CI/CD Pipeline

![GitHub Actions CI/CD pipeline run](screenshots/github-actions-pipeline.png)

### Kubernetes Workloads

![Jerney Kubernetes workloads running in Amazon EKS](screenshots/eks-workloads.png)

### AWS Infrastructure

![Jerney AWS EKS and VPC infrastructure](screenshots/aws-infrastructure.png)

## Technology Stack

| Area | Technologies |
|---|---|
| Frontend | React 18, Vite, React Router, Axios, Nginx |
| Backend | Node.js 20, Express, `pg` |
| Database | PostgreSQL 16 |
| Containers | Docker, Docker Compose |
| Cloud / Infrastructure | AWS, Amazon EKS Auto Mode, Terraform |
| Orchestration | Kubernetes |
| CI/CD and scanning | GitHub Actions, npm audit, Trivy, Checkov, Hadolint |

## Repository Structure

```text
Jerney/
├── backend/
│   ├── src/
│   │   ├── routes/             # Post and comment API routes
│   │   ├── db.js               # PostgreSQL pool and table initialization
│   │   └── index.js            # Express app and health endpoint
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   └── pages/
│   ├── Dockerfile
│   └── nginx.conf
├── k8s/
│   └── jerney.yaml             # Kubernetes resources for the application
├── terraform/
│   ├── main.tf                 # VPC and EKS cluster
│   ├── variables.tf
│   ├── outputs.tf
│   └── provider.tf
├── deploy/                     # EC2 setup script and Nginx config
├── .github/workflows/
│   └── ci-cd.yml
└── docker-compose.yml
```

## DevOps and DevSecOps Work

This project covers the delivery path from source code to a Kubernetes deployment:

1. **Containerization:** Created separate frontend and backend Docker images. The frontend uses a multi-stage build and Nginx to serve the compiled React app; the backend runs as a non-root user.
2. **Local orchestration:** Added Docker Compose to run the frontend, API, and PostgreSQL together, with a database health check and persistent data volume.
3. **Infrastructure as Code:** Used Terraform modules to define an AWS VPC across multiple availability zones and an EKS Auto Mode cluster, including private worker subnets, cluster logging, and encryption configuration.
4. **Kubernetes deployment:** Wrote manifests for the namespace, application deployments and services, PostgreSQL, encrypted EBS-backed persistent storage, resource requests/limits, readiness/liveness probes, and network policies.
5. **Continuous integration and delivery:** Built a GitHub Actions workflow that lints the app, audits dependencies, builds and publishes commit-tagged images to GHCR, scans images and IaC, and updates the Kubernetes manifest to reference the commit SHA.
6. **Security checks:** Added Trivy image scanning, Checkov Terraform/Kubernetes checks, Hadolint Dockerfile linting, restricted container settings where applicable, and least-privilege workflow permissions for package publishing.

### CI/CD Delivery Flow

```text
Push / Pull Request
        │
        ├── ESLint (frontend + backend)
        ├── npm dependency audit
        ├── Docker build (frontend + backend)
        ├── Trivy image scan
        ├── Checkov IaC scan + Hadolint
        └── On push: publish images to GHCR
                    └── update Kubernetes image tags in the manifest
```

The workflow updates image tags in Git; it does not itself run `kubectl apply` or wait for a live rollout. Apply the updated manifest to the target cluster and verify the rollout as part of deployment.

## Run Locally with Docker Compose

### Prerequisites

- Docker Desktop or Docker Engine with the Compose plugin

From the project directory:

```bash
docker compose up --build
```

Open [http://localhost](http://localhost). The frontend is served on port `80`; the API and database are available to the other Compose services. PostgreSQL data is persisted in the `pgdata` Docker volume.

To stop the services:

```bash
docker compose down
```

To stop the services and remove the local database volume:

```bash
docker compose down --volumes
```

The Compose file contains development credentials for convenience. Replace them and use a managed secrets solution before deploying to a shared or production environment.

## Run Locally without Docker

### Prerequisites

- Node.js 20+
- PostgreSQL 16+

Create a database and user in PostgreSQL, then create `backend/.env` with values for your local environment:

```dotenv
PORT=5000
DB_HOST=localhost
DB_PORT=5432
DB_USER=your_local_database_user
DB_PASSWORD=your_local_database_password
DB_NAME=jerney_db
```

Start the backend in one terminal:

```bash
cd backend
npm ci
npm start
```

Start the frontend in another terminal:

```bash
cd frontend
npm ci
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Vite proxies `/api` requests to the backend on port `5000`. The backend creates the `posts` and `comments` tables at startup.

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | API health check |
| `GET` | `/api/posts` | List posts, newest first |
| `GET` | `/api/posts/:id` | Get a post and its comments |
| `POST` | `/api/posts` | Create a post |
| `PUT` | `/api/posts/:id` | Update a post |
| `DELETE` | `/api/posts/:id` | Delete a post |
| `GET` | `/api/comments/post/:postId` | List comments for a post |
| `POST` | `/api/comments` | Add a comment |
| `DELETE` | `/api/comments/:id` | Delete a comment |

Example request to create a post:

```json
{
  "title": "My first post",
  "content": "Hello, Jerney!",
  "author": "Alex",
  "emoji": "✨"
}
```

## Deploy to AWS EKS

The deployment uses Terraform to provision an AWS VPC with public and private subnets and an EKS Auto Mode cluster. Kubernetes then runs PostgreSQL, the backend, and the frontend. This is the project's cloud deployment path; the current setup uses a NodePort and port-forward for frontend access rather than a public production ingress.

### Prerequisites

- An AWS account and configured AWS CLI credentials
- Terraform `1.5+`
- `kubectl`
- Permissions to create the required VPC, EKS, and related AWS resources

### Provision the cluster

From the `terraform` directory:

```bash
terraform init
terraform plan
terraform apply
```

The checked-in `terraform.tfvars` supplies development defaults. Review and adjust the region, cluster name, Kubernetes version, and network CIDR for your environment before applying. AWS resources may incur charges; review the plan and clean up resources when they are no longer needed.

Configure `kubectl` using the cluster name and region from your Terraform variables:

```bash
aws eks update-kubeconfig --region us-east-1 --name jerney-eks
```

### Deploy the application

From the project directory:

```bash
kubectl apply -f k8s/jerney.yaml
kubectl get pods,services -n jerney
kubectl rollout status deployment/jerney-db -n jerney
kubectl rollout status deployment/jerney-backend -n jerney
kubectl rollout status deployment/jerney-frontend -n jerney
```

The frontend service is a `NodePort`. For local access through Kubernetes, forward it to port `8080`:

```bash
kubectl port-forward -n jerney service/jerney-frontend 8080:80
```

Then open [http://localhost:8080](http://localhost:8080).

The manifest currently contains demo database credentials and fixed image references. Update both image references to images published by your workflow before applying it. Base64-encoded Kubernetes Secret values are not encrypted secrets by themselves.

### Production-readiness considerations

The project demonstrates cloud infrastructure and Kubernetes deployment, but the committed setup still needs additional work before it can safely serve a public production workload:

- Replace demo credentials with a secret-management solution; rotate any credentials that have been used outside a local demo.
- Add authentication/authorization before exposing post and comment write operations publicly.
- Configure a TLS-enabled Ingress or load balancer and domain; the current frontend uses NodePort and port-forward.
- Use a managed database or add a PostgreSQL backup, restore, upgrade, and high-availability plan. The current database is a single replica.
- Review public EKS API access, network rules, resource sizing, monitoring/alerting, and Terraform remote state with locking.
- Make security and lint checks blocking where appropriate. Several scan/lint jobs currently use `continue-on-error`, while the npm audit command also uses `|| true`.

## CI/CD Workflow

The GitHub Actions workflow in `.github/workflows/ci-cd.yml` runs on pushes and pull requests. It includes:

1. ESLint for the frontend and backend
2. npm dependency audits
3. Docker image builds, with publishing to GitHub Container Registry on pushes
4. Trivy scans of the built frontend and backend images
5. Checkov scans for Terraform and Kubernetes manifests
6. Hadolint checks for Dockerfiles
7. Kubernetes manifest image-tag update to the commit SHA on pushes

The workflow does not automatically deploy the updated manifest to EKS. Also note that some lint, image-scan, and IaC jobs are configured as non-blocking, and the dependency-audit step currently suppresses a non-zero audit exit. These checks are implemented, but are not all enforced as release gates.

## Security and Production Notes

- This project does not currently implement user authentication or authorization. Post and comment modification endpoints should not be exposed as-is for a public production service.
- Replace development credentials in Compose, Kubernetes manifests, and the EC2 helper script. Do not commit real credentials.
- Use AWS Secrets Manager, External Secrets Operator, or an equivalent solution for production secrets.
- Review cluster endpoint exposure, network rules, image tags, and Terraform state handling for your deployment requirements.

## Resume-Ready Project Summary

**Jerney — DevSecOps-Enabled Three-Tier Blog Platform | AWS, EKS, Terraform, Kubernetes, Docker, GitHub Actions**

- Provisioned AWS VPC networking and an Amazon EKS Auto Mode cluster using Terraform, and deployed a three-tier React, Express, and PostgreSQL application on Kubernetes.
- Built a GitHub Actions CI/CD workflow for code linting, dependency auditing, Docker image publishing to GHCR, commit-SHA image tagging, and Kubernetes manifest updates.
- Integrated DevSecOps checks with Trivy, Checkov, and Hadolint; configured Kubernetes health probes, resource limits, encrypted persistent storage, and network policies.
- **Before using the phrase “production deployment” on a resume, make sure you personally deployed and verified the app on a live AWS environment.** The repository currently describes a production-style deployment path, but includes demo credentials, NodePort/port-forward access, a single database replica, and no authentication.

## License

No license file is currently included in this repository.
