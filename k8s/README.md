# Local Kubernetes deployment guide (kind / minikube)

This document describes how to build the frontend and backend Docker images, load or push them into a local Kubernetes cluster (Kind or Minikube), install an Ingress controller, apply the provided manifests in `k8s/`, and verify the app is reachable at `arsam.example.com`.

WARNING: These are example commands to run locally — this file does not execute anything. Adjust image names, cluster names, and command flags to match your environment.

## Overview
- Build two images:
  - `arsam/angular-17-client:latest` (Angular static site served by nginx)
  - `arsam/spring-boot-server:latest` (Spring Boot jar)
- For local clusters you can either push these images to a registry the cluster can access, or load them directly into the cluster's container runtime (preferred for local dev).
- Install an Ingress controller (nginx ingress) and apply the `k8s/` manifests.
- Add a hosts file entry for `arsam.example.com` that resolves to the ingress IP.

## 1) Build images

From the repo root, build the frontend and backend images. Example (run in your shell):

Frontend (build in `angular-17-client`):

```bash
cd angular-17-client
# build and produce a production bundle
npm ci
npm run build -- --configuration=production
# build docker image (create image tag used in k8s manifests)
docker build -t arsam/angular-17-client:latest .
```

Backend (build in `spring-boot-server`):

```bash
cd ../spring-boot-server
# build the jar and then the docker image (Dockerfile builds with Maven inside container)
docker build -t arsam/spring-boot-server:latest .
```

Notes:
- If you prefer building the JAR locally first, you can run `./mvnw -DskipTests package` and then use a simplified Dockerfile that copies the jar into the runtime image.

## 2) Make images available to the cluster

Option A — kind (recommended for local development):

1. Create a kind cluster (if you don't already have one).
2. Load the images into the kind cluster so Pods can use them without a registry:

```bash
# Assume your cluster is named 'kind' or replace with your cluster name
kind load docker-image arsam/angular-17-client:latest --name kind
kind load docker-image arsam/spring-boot-server:latest --name kind
```

Option B — minikube:

```bash
# Use minikube's docker daemon to build, or load images into minikube
eval "$(minikube -p minikube docker-env)"
docker build -t arsam/angular-17-client:latest ./angular-17-client
docker build -t arsam/spring-boot-server:latest ./spring-boot-server
# or with newer minikube versions:
minikube image load arsam/angular-17-client:latest
minikube image load arsam/spring-boot-server:latest
```

Option C — Push to a registry (Docker Hub, GHCR, private registry):

```bash
docker tag arsam/angular-17-client:latest <registry>/arsam/angular-17-client:latest
docker tag arsam/spring-boot-server:latest <registry>/arsam/spring-boot-server:latest
docker push <registry>/arsam/angular-17-client:latest
docker push <registry>/arsam/spring-boot-server:latest
```

If you push images to a registry, update the `image:` field in the manifests under `k8s/` to point to the registry path.

## 3) Install an Ingress controller

You need an Ingress controller (example: ingress-nginx). For local clusters follow the official quickstart for your provider. Example steps (choose one):

- Install the nginx ingress controller manifest from `kubernetes/ingress-nginx` (provider manifests are available in the project's repo).
- If using Helm, install the `ingress-nginx` chart into your cluster.

After installing, confirm the ingress controller has a reachable IP/port. For kind the `ingress-nginx` controller usually exposes a NodePort on the host or uses a `NodePort` service; for minikube you can use `minikube tunnel` or `minikube service` to get access.

## 4) Apply manifests

Apply the `k8s/` manifests (namespace, backend, frontend, ingress):

```bash
# from repo root
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml
```

If your cluster cannot pull the images by the names in the manifests, update the `image:` fields accordingly or load the images (see step 2).

## 5) Hostname resolution

Map `arsam.example.com` to your ingress endpoint.

- If the ingress controller exposes a host IP on your machine (for example, `127.0.0.1` or the minikube IP), add an `/etc/hosts` entry on your local workstation:

```text
<INGRESS_IP> arsam.example.com
```

Replace `<INGRESS_IP>` with the IP/host of your ingress load balancer or node.

Note: on macOS/Linux editing `/etc/hosts` requires root. On Windows use the hosts file location and run as admin.

## 6) Verify deployment

- Check Pods and Services in the `arsam-app` namespace:

```bash
kubectl get pods -n arsam-app
kubectl get svc -n arsam-app
kubectl describe ingress -n arsam-app
```

- Confirm the ingress controller has loaded the `arsam-ingress` resource and is serving the host `arsam.example.com`.
- In a browser, open `http://arsam.example.com` — you should see the Angular app. The frontend uses a relative API path (`/api`) so API requests will be routed to the backend by the Ingress.

## 7) Common troubleshooting

- 404 on root: ensure the Ingress host in `k8s/ingress.yaml` exactly matches the host header you are using (case-sensitive). Confirm `/etc/hosts` maps the host to the ingress IP.
- API requests failing with CORS: if the API appears on a different host/port, enable CORS in the backend appropriately. With the single hostname approach CORS shouldn't be necessary.
- ImageNotFound: ensure images are available to the cluster (loaded or in registry) and `imagePullPolicy` is appropriate.
- Ingress controller not responding: check the ingress controller pods and their service type; minikube may require `minikube tunnel` or NodePort access; kind may require applying the provider-specific ingress-nginx manifest.

## Notes and recommendations

- The manifests in `k8s/` use images named `arsam/...:latest`. For reproducible deployments, tag images with a version and update manifests accordingly.
- Use a local registry (e.g., `registry:5000`) or a remote registry for CI/CD workflows.
- Add liveness/readiness probes and resource `requests`/`limits` for production-like behavior.
- For TLS in local testing, consider creating a self-signed certificate and mounting it into the Ingress, or use `cert-manager` with a staging issuer.

If you want, I can also:
- Create a `kustomization.yaml` to make environment overlays easier.
- Provide a Docker Compose + Traefik example if you'd prefer not to use Kubernetes.
