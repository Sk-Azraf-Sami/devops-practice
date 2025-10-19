# ![RealWorld Example App](logo.png)

> ### React / Express.js / Sequelize / PostgreSQL codebase containing real world examples (CRUD, auth, advanced patterns, etc) that adheres to the [RealWorld](https://realworld.io/) spec and API.

### [Demo app](https://conduit-realworld-example-app.fly.dev/)&nbsp;&nbsp;|&nbsp;&nbsp;[RealWorld Example Apps](https://codebase.show/projects/realworld?category=fullstack)&nbsp;&nbsp;|&nbsp;&nbsp;![](https://heroku-status-badges.herokuapp.com/conduit-realworld-example-app)

This codebase was created to demonstrate a fully fledged fullstack application built with **React / Express.js / Sequelize / PostgreSQL** including CRUD operations, authentication, routing, pagination, and more.

For more information on how to this works with other frontends/backends, head over to the [RealWorld](https://github.com/gothinkster/realworld) repo.

## Prerequisites

- Make sure your have a Node.js (v14 or newer) installed.
- Make sure you have your database setup.

## Installation

Install all the npm dependencies with the following command:

```bash
npm install
```

## Development

### Configuration

In the [`backend`](backend/) directory, duplicate and remane the`.env.example` file, name it `.env`, and modify it to set all the required private development environment variables.

> Optionally you can run the following command to populate your database with some dummy data:

> ```bash
> npx -w backend sequelize-cli db:seed:all
> ```

### Starting the development server

Start the development environment with the following command:

```bash
npm run dev
```

- The frontend website should be available at [http://localhost:3000/](http://localhost:3000).

- The backend API should be available at [http://localhost:3001/api](http://localhost:3001/api).

### Testing

To run the tests, run the following command:

```bash
npm test
```

## Production

The following command will build the production version of the app:

```bash
npm start
```
## Deployment Instructions

### Using Docker Compose
```bash
cd .ergomake
docker compose up --build -d
```

### Using Minikube

##### Start cluster and enable ingress controller
```bash
minikube start
minikube addons enable ingress
```

#### Build images inside Minikube's Docker
```bash
eval "$(minikube docker-env)"
docker build -t backend:latest ./backend
docker build -t frontend:latest ./frontend
```

#### Apply manifests
```bash
kubectl apply -f .ergomake/k8s-manifests.yaml
```

#### Get the IP and open the app
```bash
minikube ip
# Visit: http://<minikube-ip>/

kubectl get pods -w

# Option A — minikube service (simple)
minikube service frontend --url
# Open the printed URL (e.g., http://127.0.0.1:46143) and keep this terminal open

# Option B — Port-forward the Ingress controller
kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80
# Then open http://localhost:8080
# Test API: curl -s http://localhost:8080/api/tags

# Option C — LoadBalancer + tunnel (persistent)
kubectl -n ingress-nginx patch svc ingress-nginx-controller -p '{"spec":{"type":"LoadBalancer"}}'
sudo -E minikube tunnel
kubectl -n ingress-nginx get svc ingress-nginx-controller  # note EXTERNAL-IP
# Open http://<EXTERNAL-IP>/

# Quick sanity checks
kubectl get ingress
kubectl get ep frontend backend
```

#### If any update:
```bash
# Use Minikube’s Docker
eval "$(minikube docker-env)"

# Rebuild image with prom-client included
docker build -t backend:latest ./backend

# Restart the deployment to pick up the new image
kubectl rollout restart deploy/backend
kubectl rollout status deploy/backend
```
### Monitoring Setup
#### Pull images locally (host)
```bash
docker pull docker.io/grafana/grafana:12.2.0
docker pull quay.io/prometheus/prometheus:v3.7.1
docker pull quay.io/prometheus-operator/prometheus-config-reloader:v0.86.1
docker pull quay.io/prometheus-operator/prometheus-operator:v0.86.1
docker pull quay.io/prometheus/node-exporter:v1.9.1
docker pull registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0
docker pull quay.io/kiwigrid/k8s-sidecar:1.30.10
docker pull quay.io/prometheus/alertmanager:v0.28.1

# Load them into the Minikube node
minikube image load docker.io/grafana/grafana:12.2.0
minikube image load quay.io/prometheus/prometheus:v3.7.1
minikube image load quay.io/prometheus-operator/prometheus-config-reloader:v0.86.1
minikube image load quay.io/prometheus-operator/prometheus-operator:v0.86.1
minikube image load quay.io/prometheus/node-exporter:v1.9.1
minikube image load registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0
minikube image load quay.io/kiwigrid/k8s-sidecar:1.30.10
minikube image load quay.io/prometheus/alertmanager:v0.28.1

# Bounce the stuck pods to pick up local images
kubectl -n monitoring delete pod -l app.kubernetes.io/name=grafana
kubectl -n monitoring delete pod -l app.kubernetes.io/name=prometheus
kubectl -n monitoring delete pod -l app.kubernetes.io/name=alertmanager

# Watch until Ready
kubectl -n monitoring get pods -w

# Re-apply k8s after edits
kubectl apply -f .ergomake/k8s-manifests.yaml
kubectl apply -f .ergomake/monitoring/servicemonitors.yaml

# install helm 
sudo snap install helm --classic

# Install kube-prometheus-stack (Prometheus+Grafana)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

kubectl -n monitoring get pods
kubectl -n monitoring get svc
# Prometheus UI
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090
# Grafana UI (default admin/prom-operator)
kubectl -n monitoring port-forward svc/kps-grafana 3000:80

kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-user}' | base64 -d; echo
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```