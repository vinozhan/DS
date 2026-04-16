## Part 3 — Deployment Guide: Azure AKS + Vercel

### Why this stack

- **Azure AKS** — Managed Kubernetes. Free control plane. All 9 backend services + RabbitMQ run inside one cluster. MongoDB is offloaded to Atlas to save node RAM.
- **Vercel** — Free Vite/React hosting with global CDN, preview deploys on PRs. Better for SPAs than serving from a single k8s node.
- **Azure for Students** — $100 credit, no credit card required (.edu email). Covers ~7 weeks with a 2-node cluster.

Budget estimate: **$0** with student credits (~7 weeks). ~$60/month after credits expire (2× Standard_B2s_v2 nodes — required for 9 services + AKS system overhead).

### Architecture after deployment

```
                    ┌──────────────────────────┐
                    │  Vercel: Vite SPA         │
                    │  (frontend/)              │
                    │  https://medicare.vercel  │
                    └────────────┬──────────────┘
                                 │ HTTPS
                                 ▼
              ┌──────────────────────────────────────────┐
              │  Azure AKS Cluster (centralindia)        │
              │                                          │
              │  ┌────────────────────────────────────┐  │
              │  │ nginx Ingress Controller            │  │
              │  │ /api/auth → auth-service:3002       │  │
              │  │ /api/doctor → doctor-service:3000   │  │
              │  │ /api/appointments → appt-svc:3003   │  │
              │  │ /api/patient → patient-svc:3004     │  │
              │  │ /api/telecom → telecom-svc:3005     │  │
              │  │ /api/ai → ai-service:3006           │  │
              │  │ /api/payments → payment-svc:3007    │  │
              │  └────────────────────────────────────┘  │
              │                                          │
              │  ┌──────────────────────┐                │
              │  │ RabbitMQ Pod         │                │
              │  │ (in-cluster)         │                │
              │  └──────────────────────┘                │
              │                                          │
              │  9 backend service pods                  │
              │  (auth, doctor, appointment, patient,    │
              │   telemedicine, ai, payment,             │
              │   notification, api-gateway)             │
              └──────────────┬───────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │  MongoDB Atlas M0 (free) │
              │  7 databases per-service │
              └──────────────────────────┘
```

### Why Azure AKS over alternatives

| Factor | Azure AKS | AWS EKS | GKE Autopilot | Render |
|---|---|---|---|---|
| Control plane cost | **Free** | $73/mo | ~$73/mo | N/A |
| Student credits | **$100 (no card)** | Needs card | $300 (needs card) | $0 free tier only |
| Node cost | ~$30/mo (B2s_v2) | ~$30/mo (t3.small) | Pay-per-pod | $7/mo per service × 9 |
| MongoDB | Atlas M0 (free, saves node RAM) | Atlas / DocumentDB | Atlas | Atlas |
| RabbitMQ | **In-cluster pod** | Amazon MQ | CloudAMQP | CloudAMQP |
| Your manifests work? | **Yes, unchanged** | Yes | Yes | No (not k8s) |
| Assignment requirement | **Docker + Kubernetes ✓** | Docker + Kubernetes ✓ | Docker + Kubernetes ✓ | Docker only ✗ |

### Prerequisites

Before starting, ensure you have:

- [ ] An **Azure for Students** account — sign up at [azure.microsoft.com/en-us/free/students](https://azure.microsoft.com/en-us/free/students/) using your university email. No credit card needed. You get **$100 credit**.
- [ ] A **Vercel** account — sign up at [vercel.com](https://vercel.com) (free, connects to GitHub).
- [ ] **Azure CLI** (`az`) installed — [install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
- [ ] **kubectl** installed — `az aks install-cli` will install it for you (done in Step 2 below).
- [ ] **Docker** installed and running (for building images).
- [ ] GitHub repo pushed with latest code.
- [ ] External API keys ready: Stripe (test mode), Agora App ID + Certificate, GROQ API key, SMTP credentials (Gmail app password or SendGrid), optional Twilio.

---

### Step 1 — Pre-deployment hardening (already done if you followed Block 0)

Confirm these are in place before deploying:

- [x] **CORS lockdown**: every `main.ts` reads `CORS_ORIGINS` env var (done in Block 0).
- [x] **Health endpoints**: every service has `GET /health` (done in Block 0).
- [x] **Rate limiting**: `@nestjs/throttler` on `auth-service` (done in Block 0).
- [x] **Dockerfiles pinned**: all images use `node:20-alpine3.21` (done in Block 2).

Generate fresh production credentials now:

```bash
# Run these locally — save the output, you'll need them in Step 4
echo "JWT_SECRET=$(openssl rand -hex 64)"
echo "INTERNAL_SERVICE_KEY=$(openssl rand -hex 32)"
```

Save these values somewhere safe (password manager, not a file in the repo).

---

### Step 2 — Create the Azure AKS cluster

**2.1 — Log in to Azure:**

```bash
az login
```

This opens a browser for authentication. If you're using Azure for Students, select that subscription.

**2.2 — Verify your subscription:**

```bash
az account show --query "{name:name, id:id, state:state}" -o table
```

You should see your Azure for Students subscription with state "Enabled".

**2.3 — Create a resource group:**

Pick an allowed region. **Azure for Students restricts which regions you can use for AKS.** The resource group may create successfully in any region, but AKS creation will fail with `RequestDisallowedByAzure` if that region isn't allowed for compute resources.

Recommended region: **`centralindia`** (Mumbai — closest to Sri Lanka, lowest latency). Fallbacks: `eastus`, `westus2`, `centralus`, `northeurope`. To check your allowed regions:

```bash
az account list-locations --query "[].name" -o tsv
```

> **Troubleshooting:** If you get `RequestDisallowedByAzure` on `az aks create`, delete the resource group (`az group delete --name medicare-rg --yes`) and recreate it in a different region. Try `eastus` first — it's almost always allowed on student subscriptions.

```bash
az group create --name medicare-rg --location centralindia
```

**2.4 — Register required Azure resource providers (one-time):**

Azure for Students subscriptions don't pre-register all resource providers. Register these before creating AKS:

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.OperationsManagement

# Wait until ContainerService shows "Registered" (1–3 minutes)
az provider show --namespace Microsoft.ContainerService --query "registrationState" -o tsv
```

**2.5 — Create the AKS cluster:**

```bash
az aks create \
  --resource-group medicare-rg \
  --name medicare-aks \
  --node-count 2 \
  --node-vm-size Standard_B2s_v2 \
  --generate-ssh-keys \
  --network-plugin azure
```

This takes 3–5 minutes. Each `Standard_B2s_v2` node has 2 vCPUs and 4 GB RAM.

> **Why 2 nodes?** A single node doesn't have enough CPU for 9 backend services + RabbitMQ + AKS system pods + the app-routing nginx controller (which auto-maintains 2 replicas at 500m CPU each). See Step 6.1 for the full resource breakdown.

> **Why `Standard_B2s_v2` instead of `Standard_B2s`?** Azure is retiring v1 burstable VMs. `Standard_B2s` is unavailable in most regions on newer subscriptions. `Standard_B2s_v2` has the same specs and price (~$30/mo), just a newer generation.

**2.6 — Install kubectl and get credentials:**

```bash
# Install kubectl if you don't have it
az aks install-cli

# Download cluster credentials into ~/.kube/config
az aks get-credentials --resource-group medicare-rg --name medicare-aks

# Verify connection
kubectl get nodes
```

You should see one node in `Ready` status.

---

### Step 3 — Create Azure Container Registry (ACR) and push images

**3.1 — Create the registry:**

```bash
az acr create --resource-group medicare-rg --name medicareacr --sku Basic
```

> If `medicareacr` is taken, choose a unique name (letters + numbers only, 5–50 chars). Replace `medicareacr` everywhere below.

**3.2 — Attach ACR to AKS** (so the cluster can pull images without separate credentials):

```bash
az aks update --resource-group medicare-rg --name medicare-aks --attach-acr medicareacr
```

**3.3 — Log in to ACR:**

```bash
az acr login --name medicareacr
```

**3.4 — Build and push all 9 backend images:**

Run from the **repository root**:

```bash
# Auth service
docker build -t medicareacr.azurecr.io/healthcare/auth-service:latest ./backend/auth-service
docker push medicareacr.azurecr.io/healthcare/auth-service:latest

# Doctor service
docker build -t medicareacr.azurecr.io/healthcare/doctor-service:latest ./backend/doctor-service
docker push medicareacr.azurecr.io/healthcare/doctor-service:latest

# Appointment service
docker build -t medicareacr.azurecr.io/healthcare/appointment-service:latest ./backend/appointment-service
docker push medicareacr.azurecr.io/healthcare/appointment-service:latest

# Patient service
docker build -t medicareacr.azurecr.io/healthcare/patient-service:latest ./backend/patient-service
docker push medicareacr.azurecr.io/healthcare/patient-service:latest

# Telemedicine service
docker build -t medicareacr.azurecr.io/healthcare/telemedicine-service:latest ./backend/telemedicine-service
docker push medicareacr.azurecr.io/healthcare/telemedicine-service:latest

# AI service
docker build -t medicareacr.azurecr.io/healthcare/ai-service:latest ./backend/ai-service
docker push medicareacr.azurecr.io/healthcare/ai-service:latest

# Payment service
docker build -t medicareacr.azurecr.io/healthcare/payment-service:latest ./backend/payment-service
docker push medicareacr.azurecr.io/healthcare/payment-service:latest

# Notification service
docker build -t medicareacr.azurecr.io/healthcare/notification-service:latest ./backend/notification-service
docker push medicareacr.azurecr.io/healthcare/notification-service:latest

# API Gateway
docker build -t medicareacr.azurecr.io/healthcare/api-gateway:latest ./backend/api-gateway
docker push medicareacr.azurecr.io/healthcare/api-gateway:latest
```

**3.5 — Verify images are in ACR:**

```bash
az acr repository list --name medicareacr -o table
```

You should see 9 repositories listed.

---

### Step 4 — Configure Kubernetes secrets, config and manifests

#### 4.1 — Update image references in the manifests

Every `<service>.yaml` in `infrastructure/kubernetes/` must reference your ACR. Run from the repo root:

```bash
cd infrastructure/kubernetes

# Replace image prefix in all service yamls
sed -i 's|image: healthcare/|image: medicareacr.azurecr.io/healthcare/|g' \
  auth-service.yaml doctor-service.yaml appointment-service.yaml \
  patient-service.yaml telemedicine-service.yaml ai-service.yaml \
  payment-service.yaml notification-service.yaml

cd ../..
```

> On Windows Git Bash, `sed -i` should work. If not, use `sed -i'' 's|...|...|g'` instead. If your ACR name differs from `medicareacr`, adjust accordingly.

#### 4.2 — Database strategy: MongoDB Atlas (not in-cluster)

We use **MongoDB Atlas M0 (free tier)** instead of an in-cluster Mongo pod. This saves ~500MB RAM across the cluster for the backend services + RabbitMQ. Data also survives cluster teardowns.

**What changed in the manifests:**
- `kustomization.yaml` — `03-mongodb.yaml` is commented out (no in-cluster Mongo pod).
- `01-configmap.yaml` — global `MONGO_URI` / `MONGODB_URI` removed (each service gets its own).
- `02-secret.yaml` — 8 per-service `MONGO_URI_*` keys added, each pointing at a different Atlas database.
- Every service deployment yaml — `MONGO_URI` env var added via `secretKeyRef` to the service-specific key.

**Database-per-service mapping (7 logical databases, 8 secret keys):**

| Service | Secret Key | Atlas Database | Notes |
|---|---|---|---|
| auth-service | `MONGO_URI_AUTH` | `medicare-auth` | Users, roles, credentials |
| doctor-service | `MONGO_URI_DOCTOR` | `medicare-doctor` | Doctors, availability |
| appointment-service | `MONGO_URI_APPOINTMENT` | `medicare-appoinment` | Appointments, prescriptions |
| patient-service | `MONGO_URI_PATIENT` | `medicare-patient` | Medical records, reports |
| telemedicine-service | `MONGO_URI_TELEMEDICINE` | `medicare-appoinment` | Shared — reads appointments for Agora auth |
| payment-service | `MONGO_URI_PAYMENT` | `medicare-payment` | Stripe session history |
| notification-service | `MONGO_URI_NOTIFICATION` | `medicare-notification` | Persisted notifications |
| ai-service | `MONGO_URI_AI` | `medicare-ai` | AI consultation audit trail |

> Atlas creates databases lazily on first write — you do **not** need to pre-create them in the Atlas UI.

**Atlas Network Access:** Go to Atlas → Network Access → Add IP Address → `0.0.0.0/0` (allows all). This is required because AKS outbound IPs are dynamic. On paid Atlas tiers, use Private Endpoints instead.

#### 4.3 — Edit the secrets manifest

Open `infrastructure/kubernetes/02-secret.yaml` and fill in all values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: platform-secrets
  namespace: healthcare
type: Opaque
stringData:
  # Auth
  JWT_SECRET: '<openssl rand -hex 64 output>'
  INTERNAL_SERVICE_KEY: '<openssl rand -hex 32 output>'

  # Stripe (test mode)
  STRIPE_SECRET_KEY: 'sk_test_...'
  STRIPE_WEBHOOK_SECRET: ''
  STRIPE_PUBLISHABLE_KEY: 'pk_test_...'

  # Agora (video calls)
  AGORA_APP_ID: '<your Agora App ID>'
  AGORA_APP_CERTIFICATE: '<your Agora App Certificate>'

  # AI service
  GROQ_API_KEY: '<your GROQ API key>'

  # Email (Gmail app password or SendGrid)
  SMTP_HOST: 'smtp.gmail.com'
  SMTP_PORT: '465'
  SMTP_USER: '<your email>'
  SMTP_PASS: '<your app password>'
  SMTP_FROM: 'MediSmart Support <your-email@gmail.com>'

  # CORS — fill in after Vercel deploy (Step 7), then redeploy
  CORS_ORIGINS: ''

  # MongoDB Atlas — per-service databases
  # Replace USER:PASS@cluster.mongodb.net with your actual Atlas SRV host
  MONGO_URI_AUTH: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-auth?retryWrites=true&w=majority'
  MONGO_URI_DOCTOR: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-doctor?retryWrites=true&w=majority'
  MONGO_URI_APPOINTMENT: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-appoinment?retryWrites=true&w=majority'
  MONGO_URI_PATIENT: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-patient?retryWrites=true&w=majority'
  MONGO_URI_TELEMEDICINE: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-appoinment?retryWrites=true&w=majority'
  MONGO_URI_PAYMENT: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-payment?retryWrites=true&w=majority'
  MONGO_URI_NOTIFICATION: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-notification?retryWrites=true&w=majority'
  MONGO_URI_AI: 'mongodb+srv://USER:PASS@cluster.mongodb.net/medicare-ai?retryWrites=true&w=majority'
```

> **Do NOT commit this file with real values.** Run `git rm --cached infrastructure/kubernetes/02-secret.yaml` if it's already tracked, add it to `.gitignore`, and keep a `02-secret.example.yaml` with placeholder values in the repo instead.

#### 4.4 — Verify the ConfigMap

`infrastructure/kubernetes/01-configmap.yaml` no longer contains Mongo URIs (they're in the secret). It should contain only inter-service URLs and RabbitMQ:

```yaml
data:
  RABBITMQ_URL: amqp://rabbitmq:5672
  DOCTOR_SERVICE_URL: http://doctor-service:3000
  APPOINTMENT_SERVICE_URL: http://appointment-service:3003
```

RabbitMQ still runs as an in-cluster pod (`04-rabbitmq.yaml`). The service DNS names (`rabbitmq`, `doctor-service`, `appointment-service`) resolve automatically inside the cluster.

#### 4.5 — Verify kustomization.yaml

`03-mongodb.yaml` should be commented out since we're using Atlas:

```yaml
resources:
  - 00-namespace.yaml
  - 01-configmap.yaml
  - 02-secret.yaml
  # - 03-mongodb.yaml  # Using MongoDB Atlas instead of in-cluster pod
  - 04-rabbitmq.yaml
  - notification-service.yaml
  - auth-service.yaml
  - doctor-service.yaml
  - appointment-service.yaml
  - payment-service.yaml
  - patient-service.yaml
  - telemedicine-service.yaml
  - ai-service.yaml
  - frontend.yaml
  - ingress.yaml
```

---

### Step 5 — Enable the AKS App Routing add-on (Ingress Controller)

The Ingress routes external traffic to your backend services. Instead of the community `ingress-nginx` project (retired March 2026), we use the **AKS App Routing add-on** — a managed nginx controller maintained by Azure that requires no manual installation or upgrades.

**5.1 — Enable the add-on:**

```bash
az aks approuting enable --resource-group medicare-rg --name medicare-aks
```

This takes 1–2 minutes. Azure installs and configures the ingress controller inside the cluster automatically.

**5.2 — Verify the controller is running:**

```bash
kubectl get pods -n app-routing-system
```

You should see pods in `Running` status.

**5.3 — Get the external IP** (this is the public entry point for all your APIs):

```bash
kubectl get svc -n app-routing-system
```

The `EXTERNAL-IP` column will show a public IP. **Save this IP** — you'll need it for the Ingress host and for Vercel.

> If `EXTERNAL-IP` shows `<pending>`, wait 1–2 minutes and run the command again. Azure is provisioning a load balancer.

> **Note:** The `ingress.yaml` manifest uses `ingressClassName: webapprouting.kubernetes.azure.com` to match the AKS add-on. The `nginx.ingress.kubernetes.io/*` annotations (regex, rewrite-target) are still supported since the add-on runs nginx under the hood.

---

### Step 6 — Deploy the entire backend stack

**6.1 — Scale up to 2 nodes.**

A single B2s_v2 node (2 vCPU / 1900m allocatable) does not have enough CPU to run 9 backend services + RabbitMQ alongside the AKS system pods and the app-routing nginx controller. The breakdown:

| Component | CPU Requests |
|---|---|
| AKS system pods (coredns, metrics, csi, kube-proxy, etc.) | ~860m |
| App-routing nginx (2 replicas × 500m — auto-managed, cannot scale down) | 1000m |
| **Total before your services** | **~1860m / 1900m available** |

> **Why not scale nginx down?** The AKS app-routing add-on automatically restores 2 nginx replicas — manual `kubectl scale` is overridden within minutes.

Add a second node to double your capacity:

```bash
az aks scale --resource-group medicare-rg --name medicare-aks --node-count 2
```

This takes 2–3 minutes. Cost: ~$60/mo total (2× B2s_v2 nodes). Your $100 Azure for Students credit covers ~7 weeks.

Verify the new node is ready:

```bash
kubectl get nodes
```

You should see 2 nodes in `Ready` status, giving you **3800m total allocatable CPU** — plenty for everything.

**6.2 — Apply all manifests:**

```bash
kubectl apply -k infrastructure/kubernetes/
```

This creates:
- The `healthcare` namespace
- ConfigMap with inter-service URLs and RabbitMQ URL
- Secrets with credentials + per-service Atlas MONGO_URIs
- RabbitMQ Deployment + Service (in-cluster)
- All 9 backend Deployments + ClusterIP Services
- The Ingress routing rules

> MongoDB is **not** deployed in-cluster — services connect to Atlas directly via their `MONGO_URI` secret keys.

**6.3 — Check pod status:**

```bash
kubectl get pods -n healthcare
```

Expected startup order:
1. `rabbitmq` comes up first (~30s)
2. Backend services start next (~60s) — some may restart 1–2 times while waiting for Rabbit to accept connections. This is normal.
3. All pods should reach `Running` status with `1/1` ready within 2–3 minutes.

> **Do not use `kubectl get pods -w`** (watch mode) — it uses a long-lived TCP connection that may drop on unstable networks. Use repeated `kubectl get pods -n healthcare` instead.

If pods are stuck in `Pending`, see **Step 13 — Troubleshooting**.

**6.4 — Check for errors if any pod is in `CrashLoopBackOff`:**

```bash
# See why a pod is failing
kubectl logs <pod-name> -n healthcare --tail=50

# If the pod already restarted, check previous crash:
kubectl logs <pod-name> -n healthcare --previous --tail=50
```

Common issues:
- `MongoServerError: bad auth` → Wrong Atlas credentials in `02-secret.yaml`. Fix, reapply, and restart.
- `ECONNREFUSED` on RabbitMQ → RabbitMQ pod not ready yet. Wait and it will self-heal.
- `Error: Cannot find module` → Image wasn't built/pushed correctly. Re-run docker build + push for that service.

**6.5 — Update the Ingress host to use your real IP.**

The current `ingress.yaml` uses `healthcare.local` as the host. For public access, update it to use the nip.io wildcard DNS (free, no domain purchase needed):

```bash
# Get your Ingress external IP (from Step 5.3)
kubectl get svc -n app-routing-system
# Look for the EXTERNAL-IP on the nginx LoadBalancer row
```

Now edit `infrastructure/kubernetes/ingress.yaml` — change the host line:

```yaml
# Change this:
    - host: healthcare.local
# To this (using nip.io):
    - host: healthcare.<YOUR_INGRESS_IP>.nip.io
# Example: healthcare.20.204.188.106.nip.io
```

Apply the updated Ingress:

```bash
kubectl apply -f infrastructure/kubernetes/ingress.yaml
```

**6.6 — Verify the APIs are reachable:**

```bash
# Replace with your actual nip.io host
INGRESS_HOST="healthcare.<YOUR_INGRESS_IP>.nip.io"

# Test health endpoints
curl http://$INGRESS_HOST/api/auth/health
curl http://$INGRESS_HOST/api/doctor/health
curl http://$INGRESS_HOST/api/appointments/health
curl http://$INGRESS_HOST/api/patient/health
curl http://$INGRESS_HOST/api/telecom/health
curl http://$INGRESS_HOST/api/ai/health
curl http://$INGRESS_HOST/api/payments/health
```

Each should return `{"status":"ok","service":"<name>"}`. If any fails, check `kubectl logs` for that service.

**6.7 — Test doctor seeding:**

```bash
curl http://$INGRESS_HOST/api/doctor/doctors/search
```

Should return seeded doctor data (doctor-service seeds sample doctors on first boot).

---

### Step 7 — Deploy the SPA to Vercel

**7.1 — Log in to [vercel.com](https://vercel.com) and create a new project.**

1. Click **Add New** → **Project**.
2. **Import** your GitHub repository.
3. **Root Directory**: click "Edit" and set to `frontend`.
4. **Framework Preset**: should auto-detect as **Vite**.
5. **Build Command**: `npm run build` (default).
6. **Output Directory**: `dist` (default).

**7.2 — Set environment variables.**

In the Vercel project settings → **Environment Variables**, add:

```
VITE_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/auth
VITE_DOCTOR_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/doctor
VITE_APPOINTMENT_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/appointments
VITE_PATIENT_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/patient
VITE_TELEMEDICINE_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/telecom
VITE_AI_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/ai
VITE_PAYMENT_API_URL=http://healthcare.<YOUR_INGRESS_IP>.nip.io/api/payments
```

> Replace `<YOUR_INGRESS_IP>` with the actual IP from Step 6.4.

**7.3 — Deploy.**

Click **Deploy**. Vercel will build the SPA and give you a production URL like `https://medicare-xxxxx.vercel.app`.
https://medicare-inky.vercel.app/

**7.4 — Copy the Vercel URL and update CORS.**

Now that you know the Vercel URL, go back and update the `CORS_ORIGINS` in your Kubernetes secret:

```bash
kubectl edit secret platform-secrets -n healthcare
```

Or create a quick patch:

```bash
VERCEL_URL="https://medicare-xxxxx.vercel.app"
kubectl patch secret platform-secrets -n healthcare \
  --type merge -p "{\"stringData\":{\"CORS_ORIGINS\":\"$VERCEL_URL\"}}"
```

Then restart all deployments to pick up the new secret:

```bash
kubectl rollout restart deployment -n healthcare
```
