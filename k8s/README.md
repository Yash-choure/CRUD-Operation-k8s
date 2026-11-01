# EKS deployment

These manifests deploy the application into the `student-registration` namespace:

- MariaDB is a single-replica `StatefulSet` with an EBS-backed `gp3` PVC.
- The backend and frontend use internal `ClusterIP` Services.
- The AWS Load Balancer Controller creates one internet-facing ALB from `ingress.yaml`.
- `/api` is routed to the backend and `/` is routed to the frontend.
- HPA resources require Metrics Server to be installed in the cluster.

## Prerequisites

1. Push the backend and frontend images to a registry reachable by EKS. The
   repository currently uses `r123mahajan/backend` and `r123mahajan/frontend`.
   Replace the image names in `backend/backend.yaml` and `frontend/frontend.yaml`
   if you use a different Docker Hub account. Avoid using `latest` for
   production; use an immutable tag or digest.
2. Install and configure the AWS Load Balancer Controller with IAM permissions
   for the cluster. The controller must watch the `alb` IngressClass.
3. Ensure the Amazon EBS CSI driver is installed and its IAM role is configured.
   The `gp3` StorageClass is included for clusters that do not already provide
   one.
4. Install Metrics Server if HPA is required.

## Deploy

Build and push the two application images before deploying. The Compose file
assigns the same tags used by the Kubernetes manifests:

```powershell
docker compose build
docker login
docker push r123mahajan/backend:latest
docker push r123mahajan/frontend:latest
```

`maven:3.8.3-openjdk-17` and `node:24-alpine` are build base images used by
the application Dockerfiles. They do not need to be pushed as separate
repositories. Docker automatically transfers any required base layers when the
backend and frontend images are pushed. MariaDB is already pulled from the
public `mariadb:11.4` image and also does not need a project-owned Docker Hub
repository.

Create the database secret without committing the real password:

```powershell
kubectl create namespace student-registration
kubectl create secret generic mariadb-credentials `
  --namespace student-registration `
  --from-literal=MARIADB_ROOT_PASSWORD='<strong-password>' `
  --from-literal=DB_PASSWORD='<strong-password>'
kubectl apply -k .\k8s
```

If the namespace already exists, the first command is harmless. Do not apply
`db/secret.example.yaml`; it is documentation only. The two secret values must
match because the backend connects as the MariaDB root user in this application.

Watch rollout and find the ALB address:

```powershell
kubectl -n student-registration get pods
kubectl -n student-registration rollout status statefulset/mariadb
kubectl -n student-registration rollout status deployment/backend
kubectl -n student-registration rollout status deployment/frontend
kubectl -n student-registration get ingress student-registration
```

Open the hostname shown in the Ingress `ADDRESS`. The frontend calls the
backend at `/api`, so no public IP or backend port needs to be placed in the
frontend `.env`.

## Important production notes

- EBS is zonal and this MariaDB configuration is intentionally one replica.
  Use Amazon RDS/Aurora MariaDB for production database availability and
  backups, then remove `db/mariadb.yaml` and point `DB_HOST` at the RDS endpoint.
- The `admin` password from the Docker Compose setup must not be reused in
  production.
- `reclaimPolicy: Retain` protects the EBS volume when the PVC is deleted.
  Manage old volumes deliberately.
- Add HTTPS with an ACM certificate and an ALB HTTPS listener before exposing
  user data publicly.
