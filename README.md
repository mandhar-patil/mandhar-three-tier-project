📦 Part 1 — Application Deployment
Step 1 — Create Namespace
bashkubectl create namespace three-tier
kubectl config set-context --current --namespace=three-tier

Step 2 — Apply Secrets

⚠️ Open secrets.yaml before applying and update the username, password, and connection string with your own values.

bashkubectl apply -f secrets.yaml

# Verify secret was created
kubectl get secrets -n three-tier

💡 Why mongodb-service:27017 in the connection string?
Inside Kubernetes, every Service gets a DNS name automatically. The backend connects to mongodb-service:27017 and Kubernetes resolves it to the correct MongoDB pod — no hardcoded IPs needed.

MongoDB connection string format:
mongodb://USERNAME:PASSWORD@SERVICE-NAME:PORT/DB-NAME?authSource=admin

# Example:
mongodb://admin:mypassword@mongodb-service:27017/mydb?authSource=admin

Step 3 — Deploy MongoDB
bashkubectl apply -f mongodb.yaml

# Watch until mongodb-0 shows Running
kubectl get pods -n three-tier -w

⏳ Wait until mongodb-0 is in Running state before moving to the next step.


Step 4 — Deploy Backend

💡 Your Node.js backend must use mongoose.connect(process.env.MONGO_URI) to read the connection string from the environment.

bashkubectl apply -f backend.yaml

# Verify pod and service
kubectl get pods -n three-tier
kubectl get svc -n three-tier

Step 5 — Deploy Frontend

⚠️ Open frontend.yaml and replace the REACT_APP_BACKEND_URL value and the image name with your own values before applying.

bashkubectl apply -f frontend.yaml

# Update the backend URL with your actual server IP
kubectl set env deployment/frontend \
  REACT_APP_BACKEND_URL=http://<YOUR-SERVER-IP>:3500/api/tasks \
  -n three-tier

# Port-forward to access locally
kubectl port-forward service/frontend-service -n three-tier 3000:3000 --address=0.0.0.0 &
kubectl port-forward service/backend-service  -n three-tier 3500:3500 --address=0.0.0.0 &
Open the app at: http://<YOUR-SERVER-IP>:3000

🕸️ Part 2 — Istio Service Mesh
What is Istio?
Istio is a service mesh — it sits between all your pods and controls how they communicate. It automatically injects a small Envoy proxy sidecar into every pod, so every request passes through Envoy first.
Without Istio:
  [frontend pod]  ──────────────────▶  [backend pod]

With Istio:
  [frontend pod + envoy]  ──────────▶  [backend pod + envoy]
Traffic flow with Istio:
Browser :8080
    │
    ▼
Istio IngressGateway (NodePort/LoadBalancer)
    │
    ▼
Istio Gateway resource
    │
    ▼
Istio VirtualService
    │               │
    ▼               ▼
frontend:3000   backend:3500
                    │
                    ▼
              mongodb:27017

✅ Complete Part 1 Steps 1–5 first before proceeding with Istio.


Step 1 — Install istioctl
bashcurl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.21.0 sh -

# Add to PATH
cd istio-1.21.0
export PATH=$PWD/bin:$PATH

# Make it permanent
echo 'export PATH=$HOME/istio-1.21.0/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
istioctl version

Step 2 — Install Istio on the Cluster
bashistioctl install --set profile=demo -y

# Verify Istio pods are running
kubectl get pods -n istio-system
You should see istiod and istio-ingressgateway pods running.
ProfileDescriptionminimalCore only, no ingress gatewaydemoEverything enabled — best for learning ✅productionOptimized for real workloads

Step 3 — Enable Sidecar Injection
bash# Label the namespace — Istio will auto-inject Envoy into every pod
kubectl label namespace three-tier istio-injection=enabled

# Verify the label was applied
kubectl get ns three-tier --show-labels

💡 After labeling, every new pod in this namespace will automatically get 2 containers: your app + Envoy proxy. No YAML changes needed — Istio handles it via a webhook.


Step 4 — Restart Deployments
Restart existing pods so the Envoy sidecar gets injected into them.
bashkubectl rollout restart statefulset mongodb -n three-tier
kubectl rollout restart deployment backend -n three-tier
kubectl rollout restart deployment frontend -n three-tier

# Verify — look for 2/2 under READY column
kubectl get pods -n three-tier
Expected output:
NAME                    READY   STATUS
backend-xxxxx           2/2     Running   ← app + envoy
frontend-xxxxx          2/2     Running   ← app + envoy
mongodb-0               2/2     Running   ← app + envoy

💡 2/2 instead of 1/1 confirms Envoy was successfully injected.


Step 5 — Apply Gateway and VirtualService
bashkubectl apply -f istio-gateway.yaml

# Verify both resources were created
kubectl get gateway -n three-tier
kubectl get virtualservice -n three-tier
How routing works:
URL PathRoutes To/api/*backend-service:3500/frontend-service:3000

💡 Think of the Gateway as a bouncer at the entrance — it controls which port and protocol is allowed in. The VirtualService is the receptionist inside — it looks at your URL path and routes you to the correct service.


Step 6 — Patch IngressGateway NodePort
Map the Istio IngressGateway to NodePort 30080 so traffic from your host reaches it.
bashkubectl patch svc istio-ingressgateway -n istio-system --type='json' -p='[
  {"op":"replace","path":"/spec/ports/1/nodePort","value":30080}
]'

# Verify the NodePort is set
kubectl get svc istio-ingressgateway -n istio-system

☁️ On AWS or any cloud provider: Switch the service type to LoadBalancer instead of patching the NodePort.


Step 7 — Update Frontend Backend URL
With Istio, all traffic goes through the IngressGateway on port 8080. Update the frontend to use that port instead of 3500.
bashkubectl set env deployment/frontend \
  REACT_APP_BACKEND_URL=http://<YOUR-SERVER-IP>:8080/api/tasks \
  -n three-tier

The frontend now calls :8080/api/tasks → Istio VirtualService routes /api → backend-service:3500. The backend port is no longer exposed directly.


🌐 Access the Application
Via Port-Forward (no ingress needed)
bashkubectl port-forward service/frontend-service 3000:3000 -n three-tier --address=0.0.0.0 &
kubectl port-forward service/backend-service  3500:3500 -n three-tier --address=0.0.0.0 &
Open: http://localhost:3000
Via Istio IngressGateway
URLFrontendhttp://<YOUR-SERVER-IP>:8080Backend APIhttp://<YOUR-SERVER-IP>:8080/api/tasks

🔍 Troubleshooting
Check pod status and events
bashkubectl get pods -n three-tier
kubectl describe pod <pod-name> -n three-tier
View pod logs
bash# Single container pod
kubectl logs <pod-name> -n three-tier

# Pod with Istio sidecar (2 containers)
kubectl logs <pod-name> -c <container-name> -n three-tier
MongoDB connection errors
bash# Check if secret keys match exactly what the app expects
kubectl get secret mongo-secret -n three-tier -o yaml
Sidecar not injected (still 1/1 after Istio setup)
bash# Confirm label exists on namespace
kubectl get ns three-tier --show-labels

# Re-run rollout restarts
kubectl rollout restart statefulset mongodb -n three-tier
kubectl rollout restart deployment backend frontend -n three-tier
Istio IngressGateway not reachable
bash# Confirm NodePort is set to 30080
kubectl get svc istio-ingressgateway -n istio-system

🧹 Cleanup
bash# Remove all app resources
kubectl delete namespace three-tier
