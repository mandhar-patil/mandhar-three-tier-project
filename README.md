Part 1 — Deploy the Three-Tier App
Step 1 — Create the Kind Cluster
bash# Clone the repo
git clone https://github.com/mandhar-patil/mandhar-three-tier-project.git
cd mandhar-three-tier-project

# Create the cluster
kind create cluster --name mandhar-cluster --config kind-cluster.yaml

# Verify nodes are Ready
kubectl get nodes
Expected output:
NAME                            STATUS   ROLES
mandhar-cluster-control-plane   Ready    control-plane
mandhar-cluster-worker          Ready    <none>
mandhar-cluster-worker2         Ready    <none>

Step 2 — Create Namespace
bashkubectl create namespace three-tier
kubectl config set-context --current --namespace=three-tier

Step 3 — Apply Secrets

⚠️ Open secrets.yaml and update the username, password, and connection string before applying.

bashkubectl apply -f secrets.yaml

# Verify secret was created
kubectl get secrets -n three-tier

💡 Why mongodb-service:27017 in the connection string?
Kubernetes automatically gives every Service a DNS name inside the cluster.
The backend connects to mongodb-service:27017 and Kubernetes resolves it to the correct pod — no hardcoded IPs needed.

MongoDB connection string format:
mongodb://USERNAME:PASSWORD@SERVICE-NAME:PORT/DB-NAME?authSource=admin

# Example:
mongodb://admin:mypassword@mongodb-service:27017/taskdb?authSource=admin

Step 4 — Deploy MongoDB
bashkubectl apply -f mongodb.yaml

# Watch until mongodb-0 shows Running before proceeding
kubectl get pods -n three-tier -w

MongoDB uses a StatefulSet (not a Deployment) because it needs a stable identity and persistent storage. The service is headless (clusterIP: None) which is required for StatefulSets to work correctly.


Step 5 — Deploy Backend

💡 Your Node.js app must read the connection string from the environment variable MONGO_CONN_STR.
Make sure your code uses: mongoose.connect(process.env.MONGO_CONN_STR)

bashkubectl apply -f backend.yaml

# Verify backend pod is Running
kubectl get pods -n three-tier
kubectl get svc -n three-tier

Step 6 — Deploy Frontend

⚠️ Open frontend.yaml and replace the REACT_APP_BACKEND_URL value with your actual server IP before applying.

bashkubectl apply -f frontend.yaml

# Or update the env variable after applying (replace with your IP)
kubectl set env deployment/frontend \
  REACT_APP_BACKEND_URL=http://<YOUR-SERVER-IP>:3500/api/tasks \
  -n three-tier
To access locally via port-forward:
bashkubectl port-forward service/frontend-service 3000:3000 -n three-tier --address=0.0.0.0 &
kubectl port-forward service/backend-service  3500:3500 -n three-tier --address=0.0.0.0 &
Then open: http://localhost:3000

Part 2 — Istio Service Mesh

💡 What is Istio?
Istio is a service mesh — a network layer that sits between all your pods and controls how they talk to each other.
It automatically injects a small Envoy proxy sidecar into every pod, so every request passes through Envoy before reaching the app container.

Without Istio:
  [frontend pod]  ──────────────────>  [backend pod]

With Istio:
  [frontend pod + envoy]  ──────>  [backend pod + envoy]
Traffic flow with Istio:
Browser :8080
    │
    ▼
Istio IngressGateway (NodePort)
    │
    ▼
Istio Gateway resource
    │
    ▼
Istio VirtualService
    ├──> /api  ──>  backend-service:3500
    └──> /     ──>  frontend-service:3000

✅ Complete Part 1 Steps 1–6 and have all pods running before starting Part 2.


Step 1 — Install istioctl
bashcurl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.21.0 sh -

cd istio-1.21.0
export PATH=$PWD/bin:$PATH

# Make it permanent
echo 'export PATH=$HOME/istio-1.21.0/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
istioctl version

Step 2 — Install Istio on the Cluster
bashistioctl install --set profile=demo -y

# Verify istiod and ingressgateway pods are Running
kubectl get pods -n istio-system
ProfileUse caseminimalCore only, no ingress gatewaydemoAll features enabled — best for learning ✅productionTuned for real workloads

Step 3 — Enable Sidecar Injection
bash# Label the namespace — Istio will auto-inject Envoy into every new pod
kubectl label namespace three-tier istio-injection=enabled

# Confirm label was applied
kubectl get ns three-tier --show-labels

After labeling, every pod in this namespace gets 2 containers automatically — your app + an Envoy sidecar. No YAML changes needed; Istio handles it via a webhook.


Step 4 — Restart Deployments
Restart all workloads so the Envoy sidecar gets injected into already-running pods.
bashkubectl rollout restart statefulset mongodb    -n three-tier
kubectl rollout restart deployment  backend    -n three-tier
kubectl rollout restart deployment  frontend   -n three-tier

# Confirm 2/2 READY — that means app + envoy are both running
kubectl get pods -n three-tier
Expected output:
NAME                    READY   STATUS
backend-xxxxx           2/2     Running    <- Node.js app + Envoy
frontend-xxxxx          2/2     Running    <- React app + Envoy
mongodb-0               2/2     Running    <- MongoDB + Envoy

Step 5 — Apply Istio Gateway
The Gateway resource configures the IngressGateway pod to listen on port 80 and accept incoming HTTP traffic.
Without it, traffic reaches the IngressGateway pod but gets rejected — it has no instructions.
bashkubectl apply -f istio-gateway.yaml

# Verify Gateway is created
kubectl get gateway -n three-tier

💬 Think of the Gateway as a bouncer at a club entrance — it decides which port, protocol, and domains are allowed in. Nothing gets through without it.


Step 6 — Apply VirtualService
The VirtualService contains the routing rules — it looks at the URL path and sends traffic to the right service.
bashkubectl apply -f istio-gateway.yaml   # same file contains both Gateway + VirtualService

# Verify VirtualService is created
kubectl get virtualservice -n three-tier
Routing rules:
PathRoutes to/api/*backend-service:3500/frontend-service:3000

💬 Think of the VirtualService as the receptionist inside — once the bouncer (Gateway) lets you in, the receptionist reads your URL and sends you to the right room.


Step 7 — Patch IngressGateway NodePort
Map the Istio IngressGateway to NodePort 30080 so your host machine can reach it on port 8080.
bashkubectl patch svc istio-ingressgateway -n istio-system --type='json' -p='[
  {"op":"replace","path":"/spec/ports/1/nodePort","value":30080}
]'

# Verify the NodePort is set
kubectl get svc istio-ingressgateway -n istio-system

☁️ On AWS or any cloud provider: Change the service type to LoadBalancer instead of using NodePort.


Step 8 — Update Frontend Backend URL
With Istio, all traffic goes through the IngressGateway on port 8080.
Update the frontend so it calls the backend through Istio instead of directly.
bash# Change port 3500 → 8080 (traffic now goes through Istio)
kubectl set env deployment/frontend \
  REACT_APP_BACKEND_URL=http://<YOUR-SERVER-IP>:8080/api/tasks \
  -n three-tier

The frontend calls <ip>:8080/api/tasks → Istio VirtualService routes /api → backend-service:3500.
The backend port is no longer exposed directly.


Access the Application
Via Port-Forward (quickest, no ingress needed)
bashkubectl port-forward service/frontend-service 3000:3000 -n three-tier --address=0.0.0.0 &
kubectl port-forward service/backend-service  3500:3500 -n three-tier --address=0.0.0.0 &
Open: http://localhost:3000
Via Istio IngressGateway
WhatURLFrontendhttp://<your-server-ip>:8080Backend APIhttp://<your-server-ip>:8080/api/tasks

Cleanup
bash# Remove all app resources
kubectl delete namespace three-tier

# Delete the Kind cluster
kind delete cluster --name mandhar-cluster

Troubleshooting
Pods stuck in Pending
bashkubectl describe pod <pod-name> -n three-tier
# Read the Events section at the bottom for the root cause
MongoDB CrashLoopBackOff or connection refused
bash# Check if secret key names match what the YAML expects
kubectl get secret mongo-secret -n three-tier -o yaml

# Check MongoDB logs
kubectl logs mongodb-0 -n three-tier
Sidecar not injected — pods still show 1/1 after Istio setup
bash# Confirm the label exists
kubectl get ns three-tier --show-labels

# Re-run rollout restarts
kubectl rollout restart statefulset mongodb -n three-tier
kubectl rollout restart deployment backend frontend -n three-tier
Istio IngressGateway returns 404 or connection refused
bash# Confirm NodePort is 30080
kubectl get svc istio-ingressgateway -n istio-system

# Check Gateway and VirtualService exist in the right namespace
kubectl get gateway,virtualservice -n three-tier
View logs for any pod
bash# Single container pod
kubectl logs <pod-name> -n three-tier

# Multi-container pod (Istio injected)
kubectl logs <pod-name> -n three-tier -c backend    # app container
kubectl logs <pod-name> -n three-tier -c istio-proxy  # envoy container
