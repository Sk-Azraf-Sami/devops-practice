# Getting Started with Create React App

This project was bootstrapped with [Create React App](https://github.com/facebook/create-react-app).

## Available Scripts

In the project directory, you can run:

### `npm start`

Runs the app in the development mode.\
Open [http://localhost:3000](http://localhost:3000) to view it in your browser.

The page will reload when you make changes.\
You may also see any lint errors in the console.

### `npm test`

Launches the test runner in the interactive watch mode.\
See the section about [running tests](https://facebook.github.io/create-react-app/docs/running-tests) for more information.

### `npm run build`

Builds the app for production to the `build` folder.\
It correctly bundles React in production mode and optimizes the build for the best performance.

The build is minified and the filenames include the hashes.\
Your app is ready to be deployed!

See the section about [deployment](https://facebook.github.io/create-react-app/docs/deployment) for more information.

### `npm run eject`

**Note: this is a one-way operation. Once you `eject`, you can't go back!**

If you aren't satisfied with the build tool and configuration choices, you can `eject` at any time. This command will remove the single build dependency from your project.

Instead, it will copy all the configuration files and the transitive dependencies (webpack, Babel, ESLint, etc) right into your project so you have full control over them. All of the commands except `eject` will still work, but they will point to the copied scripts so you can tweak them. At this point you're on your own.

You don't have to ever use `eject`. The curated feature set is suitable for small and middle deployments, and you shouldn't feel obligated to use this feature. However we understand that this tool wouldn't be useful if you couldn't customize it when you are ready for it.

## Learn More

You can learn more in the [Create React App documentation](https://facebook.github.io/create-react-app/docs/getting-started).

To learn React, check out the [React documentation](https://reactjs.org/).

### Code Splitting

This section has moved here: [https://facebook.github.io/create-react-app/docs/code-splitting](https://facebook.github.io/create-react-app/docs/code-splitting)

### Analyzing the Bundle Size

This section has moved here: [https://facebook.github.io/create-react-app/docs/analyzing-the-bundle-size](https://facebook.github.io/create-react-app/docs/analyzing-the-bundle-size)

### Making a Progressive Web App

This section has moved here: [https://facebook.github.io/create-react-app/docs/making-a-progressive-web-app](https://facebook.github.io/create-react-app/docs/making-a-progressive-web-app)

### Advanced Configuration

This section has moved here: [https://facebook.github.io/create-react-app/docs/advanced-configuration](https://facebook.github.io/create-react-app/docs/advanced-configuration)

### Deployment

This section has moved here: [https://facebook.github.io/create-react-app/docs/deployment](https://facebook.github.io/create-react-app/docs/deployment)

### `npm run build` fails to minify

This section has moved here: [https://facebook.github.io/create-react-app/docs/troubleshooting#npm-run-build-fails-to-minify](https://facebook.github.io/create-react-app/docs/troubleshooting#npm-run-build-fails-to-minify)



<!-- ...existing code... -->

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