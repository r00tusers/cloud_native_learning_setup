I'll guide you through setting up Docker and Kubernetes on Ubuntu inside VirtualBox for a production-like environment.

## Prerequisites & VirtualBox Configuration

Before installing, configure your VirtualBox VM:
- **RAM**: Minimum 4GB (8GB+ recommended for production)
- **CPUs**: At least 2 cores (4+ recommended)
- **Disk**: 40GB+ available space
- **Network**: Bridged Adapter or NAT with port forwarding
- **Enable nested virtualization** if your host supports it

## 1. Install Docker

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group (logout/login required)
sudo usermod -aG docker $USER

# Verify installation
docker --version
```

## 2. Configure Docker for Kubernetes

```bash
# Configure Docker daemon for Kubernetes
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 3. Disable Swap (Required for Kubernetes)

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify
free -h
```

## 4. Enable Kernel Modules & Network Settings

```bash
# Load required modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## 5. Install Kubernetes Components

You have two main options: **kubeadm** (manual cluster) or **k3s/minikube** (simplified). For production-like setup, I recommend **kubeadm**:

### Option A: kubeadm (Full Kubernetes)

```bash
# Add Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialize cluster (single-node for VirtualBox)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Pod network (Flannel)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Allow scheduling on control plane (single-node setup)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Option B: K3s (Lightweight Kubernetes - Good for VirtualBox)

```bash
# Install K3s
curl -sfL https://get.k3s.io | sh -

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
```

## 6. Essential Production Tools to Install

### Container Runtime & Management
```bash
# Already have Docker, but also install:
sudo apt install -y containerd

# Helm (Kubernetes package manager)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Monitoring & Observability
```bash
# Install Prometheus & Grafana via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

### Ingress Controller
```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### Storage
```bash
# Install local-path-provisioner for dynamic storage
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### Additional Essential Tools
```bash
# kubectl plugins manager
curl -LO https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-linux_amd64.tar.gz
tar zxvf krew-linux_amd64.tar.gz
./krew-linux_amd64 install krew
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc

# k9s (Kubernetes CLI dashboard)
curl -sS https://webinstall.dev/k9s | bash

# Lens or k9s for cluster management
# kubectx & kubens for context switching
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

## 7. Security & Production Essentials

```bash
# Install Falco (runtime security)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace

# Install cert-manager (TLS certificate management)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Setup RBAC and network policies
# (Create these based on your specific needs)
```

## 8. Verification

```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes
kubectl get pods -A

# Check Docker
docker ps
systemctl status docker

# Check versions
kubeadm version
kubectl version --client
docker --version
```

## Production Considerations for VirtualBox

1. **Snapshots**: Take VM snapshots before major changes
2. **Resource limits**: Monitor CPU/RAM usage
3. **Networking**: Use bridged adapter for external access
4. **Backups**: Regularly backup `/etc/kubernetes` and persistent volumes
5. **Load balancing**: Consider MetalLB for LoadBalancer services
6. **CI/CD**: Install Jenkins, GitLab Runner, or ArgoCD
7. **Service mesh**: Consider Istio or Linkerd for advanced networking
8. **Logging**: Add ELK/EFK stack or Loki for log aggregation

Would you like me to elaborate on any specific part or help you set up additional components?

# Django Application Deployment with PostgreSQL
apiVersion: v1
kind: Namespace
metadata:
  name: django-apps
  labels:
    istio-injection: enabled
---
# ConfigMap for Django settings
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: django-apps
data:
  DJANGO_SETTINGS_MODULE: "myapp.settings"
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
  namespace: django-apps
type: Opaque
stringData:
  DATABASE_PASSWORD: "changeme123"
  DJANGO_SECRET_KEY: "your-secret-key-here-change-in-production"
---
# PostgreSQL StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: django-apps
spec:
  ports:
    - port: 5432
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: django-apps
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        version: v1
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: djangodb
        - name: POSTGRES_USER
          value: django
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: DATABASE_PASSWORD
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
---
# Redis Deployment
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: django-apps
spec:
  ports:
    - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: django-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        version: v1
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
# Django App 1 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app1
  namespace: django-apps
  labels:
    app: django-app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app1
  template:
    metadata:
      labels:
        app: django-app1
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: django
        image: your-registry/django-app1:latest
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: DATABASE_NAME
          value: "djangodb"
        - name: DATABASE_USER
          value: "django"
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: DATABASE_PASSWORD
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: django-config
              key: DATABASE_HOST
        - name: DJANGO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: DJANGO_SECRET_KEY
        livenessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready/
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
# Service for Django App 1
apiVersion: v1
kind: Service
metadata:
  name: django-app1-service
  namespace: django-apps
  labels:
    app: django-app1
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  selector:
    app: django-app1
---
# Repeat similar structure for django-app2, app3, app4
# (Just change the names and labels accordingly)

# Istio Gateway Configuration
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "django-app1.local"
    - "django-app2.local"
    - "django-app3.local"
    - "django-app4.local"
    - "fastapi-app1.local"
    - "fastapi-app2.local"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: apps-tls-cert
    hosts:
    - "django-app1.local"
    - "django-app2.local"
    - "django-app3.local"
    - "django-app4.local"
    - "fastapi-app1.local"
    - "fastapi-app2.local"
---
# VirtualService for Django App 1
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: django-app1-vs
  namespace: django-apps
spec:
  hosts:
  - "django-app1.local"
  gateways:
  - istio-system/main-gateway
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: django-app1-service
        port:
          number: 80
      weight: 100
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
---
# VirtualService for FastAPI App 1
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fastapi-app1-vs
  namespace: fastapi-apps
spec:
  hosts:
  - "fastapi-app1.local"
  gateways:
  - istio-system/main-gateway
  http:
  - match:
    - uri:
        prefix: "/api"
    route:
    - destination:
        host: fastapi-app1-service
        port:
          number: 80
      weight: 90
    - destination:
        host: fastapi-app1-service
        subset: canary
        port:
          number: 80
      weight: 10
    timeout: 15s
    retries:
      attempts: 3
      perTryTimeout: 5s
---
# DestinationRule for FastAPI (traffic management)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fastapi-app1-dr
  namespace: fastapi-apps
spec:
  host: fastapi-app1-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    loadBalancer:
      simple: LEAST_REQUEST
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: canary
    labels:
      version: canary
---
# ServiceEntry for external database (if needed)
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-postgres
  namespace: django-apps
spec:
  hosts:
  - external-db.example.com
  ports:
  - number: 5432
    name: postgres
    protocol: TCP
  location: MESH_EXTERNAL
  resolution: DNS
---
# PeerAuthentication for mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# AuthorizationPolicy for Django apps
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: django-authz
  namespace: django-apps
spec:
  selector:
    matchLabels:
      app: django-app1
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
    to:
    - operation:
        methods: ["GET", "POST", "PUT", "DELETE"]
        paths: ["/*"]

# Cilium Network Policy for Django Apps
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: django-app-policy
  namespace: django-apps
spec:
  endpointSelector:
    matchLabels:
      app: django-app1
  ingress:
  - fromEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: istio-system
        app: istio-ingressgateway
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
  - fromEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: fastapi-apps
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
  egress:
  # Allow DNS
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
      rules:
        dns:
        - matchPattern: "*"
  # Allow PostgreSQL
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: django-apps
        app: postgres
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
  # Allow Redis
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: django-apps
        app: redis
    toPorts:
    - ports:
      - port: "6379"
        protocol: TCP
  # Allow HTTPS to external APIs
  - toFQDNs:
    - matchPattern: "*.googleapis.com"
    - matchPattern: "*.github.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
---
# Cilium Network Policy for FastAPI Apps
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: fastapi-app-policy
  namespace: fastapi-apps
spec:
  endpointSelector:
    matchLabels:
      app: fastapi-app1
  ingress:
  - fromEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: istio-system
        app: istio-ingressgateway
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
  - fromEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: django-apps
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
  egress:
  # Allow DNS
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
  # Allow communication to Django apps
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: django-apps
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
  # Allow external HTTPS
  - toCIDR:
    - 0.0.0.0/0
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
---
# L7 HTTP Policy for rate limiting
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: http-rate-limit
  namespace: fastapi-apps
spec:
  endpointSelector:
    matchLabels:
      app: fastapi-app1
  ingress:
  - fromEndpoints:
    - {}
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"
        - method: "POST"
          path: "/api/.*"
          headers:
          - "Content-Type: application/json"
---
# Deny all default policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: django-apps
spec:
  endpointSelector: {}
  ingress:
  - {}
  egress:
  - {}
---
# Cluster-wide policy for monitoring namespace
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-prometheus-scraping
spec:
  endpointSelector: {}
  ingress:
  - fromEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: monitoring
        app.kubernetes.io/name: prometheus
    toPorts:
    - ports:
      - port: "8000"
        protocol: TCP
      - port: "9090"
        protocol: TCP

Perfect! Now let me create comprehensive guides for setting up the entire production-like infrastructure:

## **Complete Production Cluster Setup Guide**

### **Phase 1: Replace Docker with Containerd + Install Cilium (CNI with eBPF)**

Cilium is a modern CNI that uses eBPF for networking, security, and observability. It's better than Flannel for production.

```bash
# 1. Remove Docker as Kubernetes runtime (keep Docker for building images)
sudo systemctl stop kubelet

# 2. Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Edit containerd config for systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

# 3. Initialize Kubernetes WITHOUT a CNI (we'll use Cilium)
sudo kubeadm reset -f  # If already initialized
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 4. Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# 5. Install Cilium with eBPF features enabled
cilium install --version 1.14.5 \
  --set kubeProxyReplacement=strict \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true

# 6. Verify Cilium installation
cilium status --wait
cilium connectivity test  # This will take several minutes

# 7. Allow scheduling on control plane
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### **Phase 2: Install Istio Service Mesh**

```bash
# 1. Download Istio
cd ~
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH
echo 'export PATH="$HOME/istio-1.20.0/bin:$PATH"' >> ~/.bashrc

# 2. Install Istio with production profile
istioctl install --set profile=production -y

# 3. Enable Istio sidecar injection for namespaces
kubectl label namespace default istio-injection=enabled

# 4. Install Istio addons (Kiali, Jaeger, Prometheus, Grafana)
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
kubectl apply -f samples/addons/kiali.yaml

# 5. Expose Kiali dashboard
kubectl patch service kiali -n istio-system -p '{"spec":{"type":"NodePort"}}'

# Get Kiali URL
kubectl get svc kiali -n istio-system
```

### **Phase 3: Install Monitoring Stack (Prometheus + Grafana)**

```bash
# 1. Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. Create monitoring namespace
kubectl create namespace monitoring

# 3. Install Prometheus Operator Stack (includes Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort

# 4. Install Prometheus metrics for Cilium
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.14/examples/kubernetes/addons/prometheus/monitoring-example.yaml

# 5. Get Grafana URL
kubectl get svc -n monitoring prometheus-grafana

# Access Grafana: http://<node-ip>:<nodeport>
# Default login: admin / admin123
```

### **Phase 4: Install Falco (Runtime Security)**

```bash
# 1. Add Falco Helm repo
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# 2. Install Falco with eBPF driver
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=ebpf \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# 3. Expose Falco Sidekick UI
kubectl patch service falco-falcosidekick-ui -n falco -p '{"spec":{"type":"NodePort"}}'

# 4. Get Falco UI URL
kubectl get svc -n falco falco-falcosidekick-ui

# 5. View Falco logs
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

### **Phase 5: Container Image Setup**

Create Dockerfiles for your applications:

**Django Dockerfile:**
```dockerfile
# Dockerfile.django
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt gunicorn psycopg2-binary redis django-prometheus

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 django && chown -R django:django /app
USER django

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "myapp.wsgi:application"]
```

**FastAPI Dockerfile:**
```dockerfile
# Dockerfile.fastapi
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt uvicorn fastapi prometheus-client

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 fastapi && chown -R fastapi:fastapi /app
USER fastapi

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**Build and push images:**
```bash
# Build Django images
docker build -t localhost:5000/django-app1:latest -f Dockerfile.django ./django-app1
docker build -t localhost:5000/django-app2:latest -f Dockerfile.django ./django-app2
docker build -t localhost:5000/django-app3:latest -f Dockerfile.django ./django-app3
docker build -t localhost:5000/django-app4:latest -f Dockerfile.django ./django-app4

# Build FastAPI images
docker build -t localhost:5000/fastapi-app1:latest -f Dockerfile.fastapi ./fastapi-app1
docker build -t localhost:5000/fastapi-app2:latest -f Dockerfile.fastapi ./fastapi-app2

# Setup local registry (for VirtualBox environment)
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Push images
docker push localhost:5000/django-app1:latest
docker push localhost:5000/django-app2:latest
docker push localhost:5000/django-app3:latest
docker push localhost:5000/django-app4:latest
docker push localhost:5000/fastapi-app1:latest
docker push localhost:5000/fastapi-app2:latest
```

### **Phase 6: Deploy Applications**

```bash
# Update image references in the YAML files to use your registry
# Change: your-registry/django-app1:latest 
# To: localhost:5000/django-app1:latest

# Apply Django deployments
kubectl apply -f django-deployments.yaml

# Apply FastAPI deployments
kubectl apply -f fastapi-deployments.yaml

# Verify deployments
kubectl get pods -n django-apps
kubectl get pods -n fastapi-apps

# Check Istio sidecar injection
kubectl get pods -n django-apps -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

### **Phase 7: Setup Ingress with TLS**### **Phase 8: Observability with Hubble (Cilium's eBPF Observability)**

```bash
# 1. Install Hubble CLI
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}

# 2. Enable Hubble UI port forwarding
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 --address 0.0.0.0 &

# Access Hubble UI at: http://your-vm-ip:12000

# 3. Enable Hubble relay port forwarding for CLI
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 --address 0.0.0.0 &

# 4. Use Hubble CLI to observe traffic
hubble observe --namespace django-apps
hubble observe --namespace fastapi-apps --follow

# 5. Monitor specific flows
hubble observe --from-namespace django-apps --to-namespace fastapi-apps
hubble observe --verdict DROPPED  # See dropped packets

# 6. Network policy testing
hubble observe --type trace:to-endpoint --verdict DROPPED
```

### **Phase 9: Network Policies with Cilium**### **Phase 10: Setup Grafana Dashboards**

```bash
# 1. Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address 0.0.0.0 &

# Login: admin / admin123
# Navigate to: http://your-vm-ip:3000

# 2. Import pre-built dashboards
# Go to Dashboards → Import

# Import these dashboard IDs:
# - 15513: Cilium Metrics
# - 7636: Cilium Operator
# - 7645: Istio Control Plane
# - 7639: Istio Performance
# - 13770: Kubernetes Cluster Monitoring
# - 6417: Kubernetes Pods
# - 11074: Node Exporter Full

# 3. Setup data sources
# Prometheus is already configured
# Add Loki for logs (optional):
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true

# 4. Add Hubble as data source
# In Grafana: Configuration → Data Sources → Add → Prometheus
# URL: http://hubble-metrics.kube-system.svc.cluster.local:9965
```

### **Phase 11: Load Testing & Observability**

```bash
# 1. Install load testing tools
kubectl create namespace load-test

# Deploy load testing pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: load-tester
  namespace: load-test
spec:
  containers:
  - name: load-tester
    image: williamyeh/hey:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
EOF

# 2. Run load tests
kubectl exec -it load-tester -n load-test -- sh

# Inside the pod, run tests:
hey -z 60s -c 50 -q 10 http://django-app1-service.django-apps.svc.cluster.local
hey -z 60s -c 100 -q 20 http://fastapi-app1-service.fastapi-apps.svc.cluster.local

# 3. Monitor in real-time
# Terminal 1: Watch Hubble
hubble observe --namespace django-apps --follow

# Terminal 2: Watch Prometheus metrics
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Terminal 3: Watch Falco alerts
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# Terminal 4: Watch pods autoscaling
watch kubectl get hpa -A
```

### **Phase 12: eBPF Programs Exploration**

```bash
# 1. Check loaded eBPF programs
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list
kubectl exec -n kube-system ds/cilium -- cilium bpf ct list global
kubectl exec -n kube-system ds/cilium -- cilium bpf policy get

# 2. Install bpftool for deeper inspection
sudo apt install -y linux-tools-common linux-tools-generic
sudo bpftool prog list
sudo bpftool map list

# 3. View Cilium eBPF stats
kubectl exec -n kube-system ds/cilium -- cilium status --verbose
kubectl exec -n kube-system ds/cilium -- cilium bpf metrics list

# 4. Monitor eBPF events
kubectl exec -n kube-system ds/cilium -- cilium monitor

# 5. Check Falco eBPF driver
kubectl exec -n falco ds/falco -- falco --list
kubectl logs -n falco -l app.kubernetes.io/name=falco | grep -i ebpf
```

### **Phase 13: Tracing with Jaeger**

```bash
# 1. Enable tracing in applications
# Add to Django settings.py:
# pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-django

# 2. Access Jaeger UI
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686 --address 0.0.0.0 &

# Access at: http://your-vm-ip:16686

# 3. Generate traced traffic
# Make requests to your apps and view traces in Jaeger

# 4. Configure sampling rates
kubectl edit configmap istio -n istio-system
# Add: tracing.sampling: 100  # 100% sampling for testing
```

### **Phase 14: Backup and Disaster Recovery**

```bash
# 1. Install Velero for backup
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# 2. Setup MinIO for backup storage
kubectl create namespace velero
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/velero/main/examples/minio/00-minio-deployment.yaml

# 3. Configure Velero
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.8.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000

# 4. Create backup schedule
velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces django-apps,fastapi-apps
```

### **Complete Dashboard Access Summary**

```bash
# Port forward all dashboards
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address 0.0.0.0 &
kubectl port-forward -n istio-system svc/kiali 20001:20001 --address 0.0.0.0 &
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686 --address 0.0.0.0 &
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 --address 0.0.0.0 &
kubectl port-forward -n falco svc/falco-falcosidekick-ui 2802:2802 --address 0.0.0.0 &
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 --address 0.0.0.0 &

echo "Access dashboards at:"
echo "Grafana: http://$(hostname -I | awk '{print $1}'):3000 (admin/admin123)"
echo "Kiali: http://$(hostname -I | awk '{print $1}'):20001"
echo "Jaeger: http://$(hostname -I | awk '{print $1}'):16686"
echo "Hubble: http://$(hostname -I | awk '{print $1}'):12000"
echo "Falco: http://$(hostname -I | awk '{print $1}'):2802"
echo "Prometheus: http://$(hostname -I | awk '{print $1}'):9090"
```

### **Key Learning Areas Covered:**

1. **eBPF**: Cilium uses eBPF for networking, load balancing, security
2. **Service Mesh**: Istio provides traffic management, security, observability
3. **Monitoring**: Prometheus + Grafana for metrics
4. **Observability**: Hubble for network flows, Jaeger for distributed tracing
5. **Security**: Falco for runtime security, mTLS with Istio
6. **Network Policies**: Layer 3/4/7 policies with Cilium
7. **Auto-scaling**: HPA based on CPU/memory
8. **Load Balancing**: Multiple algorithms (Round Robin, Least Request)
9. **Traffic Management**: Canary deployments, circuit breakers, retries
10. **Backup/Restore**: Velero for disaster recovery

This setup gives you a production-grade Kubernetes cluster with all modern cloud-native tools! Let me know which area you want to dive deeper into.

You need to install Kubernetes components first! Let me guide you through the complete installation from scratch.

## **Complete Installation from Scratch**

### **Step 1: System Preparation**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg2 lsb-release

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is off
free -h
```

### **Step 2: Install containerd (Container Runtime)**

```bash
# Install containerd
sudo apt install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

### **Step 3: Configure Kernel Modules and Network Settings**

```bash
# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system

# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify sysctl settings
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### **Step 4: Install Kubernetes Components (kubeadm, kubelet, kubectl)**

```bash
# Add Kubernetes GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt update

# Install Kubernetes components
sudo apt install -y kubelet kubeadm kubectl

# Hold packages at current version
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet

# Verify installation
kubeadm version
kubectl version --client
kubelet --version
```

### **Step 5: Initialize Kubernetes Cluster (WITHOUT CNI for now)**

```bash
# Initialize cluster without kube-proxy (we'll use Cilium for everything)
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --skip-phases=addon/kube-proxy \
  --cri-socket=unix:///run/containerd/containerd.sock

# If the above command succeeds, you'll see output like:
# "Your Kubernetes control-plane has initialized successfully!"
```

### **Step 6: Configure kubectl for Your User**

```bash
# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify kubectl works
kubectl get nodes
# Should show: NotReady (because CNI is not installed yet)

kubectl get pods -A
# Should show control plane pods
```

### **Step 7: Allow Scheduling on Control Plane (Single Node Setup)**

```bash
# Remove taint from control plane node to allow pod scheduling
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Verify
kubectl describe node | grep Taints
# Should show: Taints: <none>
```

### **Step 8: Install Cilium (CNI with eBPF)**

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Verify Cilium CLI
cilium version --client

# Install Cilium with eBPF features
cilium install --version 1.14.5 \
  --set kubeProxyReplacement=strict \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true

# Wait for Cilium to be ready (this may take 2-3 minutes)
cilium status --wait

# Verify cluster is now ready
kubectl get nodes
# Should show: Ready

kubectl get pods -A
# All pods should be Running
```

### **Step 9: Verify Cilium Installation**

```bash
# Check Cilium status
cilium status

# Run connectivity test (optional, takes 5-10 minutes)
cilium connectivity test

# Check eBPF programs loaded
kubectl exec -n kube-system ds/cilium -- cilium status --verbose
```

### **Step 10: Install Docker (for Building Images)**

```bash
# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER

# Note: You need to logout and login for group changes to take effect
# For now, you can use: newgrp docker

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify Docker
docker --version
docker run hello-world
```

### **Step 11: Setup Local Docker Registry**

```bash
# Create registry container
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Verify registry is running
docker ps | grep registry

# Configure insecure registry in containerd
sudo mkdir -p /etc/containerd/certs.d/localhost:5000
cat <<EOF | sudo tee /etc/containerd/certs.d/localhost:5000/hosts.toml
[host."http://localhost:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
EOF

# Restart containerd
sudo systemctl restart containerd
```

### **Step 12: Quick Cluster Verification**

```bash
# Check all components
echo "=== Node Status ==="
kubectl get nodes -o wide

echo "=== All Pods ==="
kubectl get pods -A

echo "=== Cilium Status ==="
cilium status

echo "=== Container Runtime ==="
kubectl get nodes -o wide | awk '{print $1,$8}'

# Create a test deployment
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=ClusterIP

# Wait a moment
sleep 10

# Check if it's running
kubectl get pods
kubectl get svc

# Test connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- nginx
```

### **Troubleshooting Common Issues**

If you encounter issues:

```bash
# Check kubelet logs
sudo journalctl -u kubelet -f

# Check containerd logs
sudo journalctl -u containerd -f

# Reset cluster if needed
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Then start over from Step 5
```

### **What's Next?**

Once you see all pods running and the node shows "Ready", you can proceed with:
1. Installing Istio (Phase 2 from previous message)
2. Installing Prometheus/Grafana (Phase 3)
3. Deploying your Django and FastAPI applications

**Current Status Check:**
```bash
# Run this to see your cluster status
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
cilium status
```

Let me know the output and we'll continue with the next steps! If you see any errors, share them and I'll help you fix them.

Good progress! Your Cilium core networking is working perfectly. The Hubble UI and Relay pods are pending, which is usually due to resource constraints in VirtualBox or missing image pulls. Let's fix this and move forward.

## **Fix Hubble Issues and Continue**

```bash
# Check why Hubble pods are pending
kubectl describe pod -n kube-system -l k8s-app=hubble-relay
kubectl describe pod -n kube-system -l k8s-app=hubble-ui

# Most likely it's image pull issues. Let's check:
kubectl get events -n kube-system --sort-by='.lastTimestamp' | tail -20

# For now, Hubble UI is optional. Let's verify the core cluster works:
kubectl get nodes
kubectl get pods -A | grep -v hubble
```

Since Cilium core is working (which is the most important part), let's proceed with setting up your applications!

## **Deploy Your Applications Now**

### **Step 1: Create Sample Django App Structure**

```bash
# Create project directory
mkdir -p ~/k8s-apps && cd ~/k8s-apps
mkdir -p django-apps/{app1,app2,app3,app4}
mkdir -p fastapi-apps/{app1,app2}
```

### **Step 2: Create Simple Django Application**

```bash
# Create Django app1
cat > django-apps/app1/requirements.txt <<EOF
Django==4.2.7
gunicorn==21.2.0
psycopg2-binary==2.9.9
redis==5.0.1
django-prometheus==2.3.1
EOF

cat > django-apps/app1/app.py <<'EOF'
from django.conf import settings
from django.core.wsgi import get_wsgi_application
from django.http import JsonResponse
from django.urls import path
import os

# Minimal Django settings
settings.configure(
    DEBUG=False,
    SECRET_KEY=os.environ.get('DJANGO_SECRET_KEY', 'dev-secret-key-change-me'),
    ALLOWED_HOSTS=['*'],
    ROOT_URLCONF=__name__,
    MIDDLEWARE=[
        'django.middleware.common.CommonMiddleware',
    ],
    DATABASES={
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': os.environ.get('DATABASE_NAME', 'djangodb'),
            'USER': os.environ.get('DATABASE_USER', 'django'),
            'PASSWORD': os.environ.get('DATABASE_PASSWORD', 'changeme123'),
            'HOST': os.environ.get('DATABASE_HOST', 'postgres-service'),
            'PORT': os.environ.get('DATABASE_PORT', '5432'),
        }
    }
)

def health(request):
    return JsonResponse({'status': 'healthy', 'app': 'django-app1'})

def ready(request):
    return JsonResponse({'status': 'ready', 'app': 'django-app1'})

def index(request):
    return JsonResponse({
        'app': 'django-app1',
        'version': '1.0',
        'message': 'Hello from Django App 1!'
    })

urlpatterns = [
    path('', index),
    path('health/', health),
    path('ready/', ready),
]

application = get_wsgi_application()
EOF

cat > django-apps/app1/Dockerfile <<'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd -m -u 1000 django && chown -R django:django /app
USER django

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:application"]
EOF

# Copy to other Django apps (with modifications)
for i in 2 3 4; do
  cp -r django-apps/app1/* django-apps/app$i/
  sed -i "s/django-app1/django-app$i/g" django-apps/app$i/app.py
done
```

### **Step 3: Create Simple FastAPI Application**

```bash
# Create FastAPI app1
cat > fastapi-apps/app1/requirements.txt <<EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
prometheus-client==0.19.0
EOF

cat > fastapi-apps/app1/main.py <<'EOF'
from fastapi import FastAPI
from fastapi.responses import JSONResponse
import os

app = FastAPI(title="FastAPI App 1")

@app.get("/")
async def root():
    return {
        "app": "fastapi-app1",
        "version": "1.0",
        "message": "Hello from FastAPI App 1!"
    }

@app.get("/health")
async def health():
    return {"status": "healthy", "app": "fastapi-app1"}

@app.get("/api/data")
async def get_data():
    return {
        "data": [1, 2, 3, 4, 5],
        "source": "fastapi-app1"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF

cat > fastapi-apps/app1/Dockerfile <<'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

RUN useradd -m -u 1000 fastapi && chown -R fastapi:fastapi /app
USER fastapi

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
EOF

# Copy to FastAPI app2
cp -r fastapi-apps/app1/* fastapi-apps/app2/
sed -i "s/fastapi-app1/fastapi-app2/g" fastapi-apps/app2/main.py
```

### **Step 4: Build and Push Docker Images**

```bash
cd ~/k8s-apps

# Build Django images
for i in 1 2 3 4; do
  echo "Building django-app$i..."
  docker build -t localhost:5000/django-app$i:latest django-apps/app$i/
  docker push localhost:5000/django-app$i:latest
done

# Build FastAPI images
for i in 1 2; do
  echo "Building fastapi-app$i..."
  docker build -t localhost:5000/fastapi-app$i:latest fastapi-apps/app$i/
  docker push localhost:5000/fastapi-app$i:latest
done

# Verify images in registry
curl http://localhost:5000/v2/_catalog
```

### **Step 5: Create Kubernetes Manifests**

```bash
# Create complete deployment file
cat > ~/k8s-apps/complete-deployment.yaml <<'EOF'
# Namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: django-apps
  labels:
    name: django-apps
---
apiVersion: v1
kind: Namespace
metadata:
  name: fastapi-apps
  labels:
    name: fastapi-apps
---
# PostgreSQL for Django
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: django-apps
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: django-apps
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: djangodb
        - name: POSTGRES_USER
          value: django
        - name: POSTGRES_PASSWORD
          value: changeme123
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
---
# Redis for Django
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: django-apps
spec:
  ports:
    - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: django-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
# Django App 1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app1
  namespace: django-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app1
  template:
    metadata:
      labels:
        app: django-app1
        version: v1
    spec:
      containers:
      - name: django
        image: localhost:5000/django-app1:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_NAME
          value: "djangodb"
        - name: DATABASE_USER
          value: "django"
        - name: DATABASE_PASSWORD
          value: "changeme123"
        - name: DATABASE_HOST
          value: "postgres-service"
        - name: DJANGO_SECRET_KEY
          value: "production-secret-key-change-me"
        livenessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready/
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: django-app1-service
  namespace: django-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30001
  selector:
    app: django-app1
---
# Django App 2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app2
  namespace: django-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app2
  template:
    metadata:
      labels:
        app: django-app2
        version: v1
    spec:
      containers:
      - name: django
        image: localhost:5000/django-app2:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_NAME
          value: "djangodb"
        - name: DATABASE_USER
          value: "django"
        - name: DATABASE_PASSWORD
          value: "changeme123"
        - name: DATABASE_HOST
          value: "postgres-service"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: django-app2-service
  namespace: django-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30002
  selector:
    app: django-app2
---
# Django App 3
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app3
  namespace: django-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app3
  template:
    metadata:
      labels:
        app: django-app3
        version: v1
    spec:
      containers:
      - name: django
        image: localhost:5000/django-app3:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_NAME
          value: "djangodb"
        - name: DATABASE_USER
          value: "django"
        - name: DATABASE_PASSWORD
          value: "changeme123"
        - name: DATABASE_HOST
          value: "postgres-service"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: django-app3-service
  namespace: django-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30003
  selector:
    app: django-app3
---
# Django App 4
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app4
  namespace: django-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app4
  template:
    metadata:
      labels:
        app: django-app4
        version: v1
    spec:
      containers:
      - name: django
        image: localhost:5000/django-app4:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_NAME
          value: "djangodb"
        - name: DATABASE_USER
          value: "django"
        - name: DATABASE_PASSWORD
          value: "changeme123"
        - name: DATABASE_HOST
          value: "postgres-service"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: django-app4-service
  namespace: django-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30004
  selector:
    app: django-app4
---
# FastAPI App 1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app1
  namespace: fastapi-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-app1
  template:
    metadata:
      labels:
        app: fastapi-app1
        version: v1
    spec:
      containers:
      - name: fastapi
        image: localhost:5000/fastapi-app1:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-app1-service
  namespace: fastapi-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30011
  selector:
    app: fastapi-app1
---
# FastAPI App 2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app2
  namespace: fastapi-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-app2
  template:
    metadata:
      labels:
        app: fastapi-app2
        version: v1
    spec:
      containers:
      - name: fastapi
        image: localhost:5000/fastapi-app2:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-app2-service
  namespace: fastapi-apps
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30012
  selector:
    app: fastapi-app2
EOF
```

### **Step 6: Deploy Everything**

```bash
# Apply the deployment
kubectl apply -f ~/k8s-apps/complete-deployment.yaml

# Watch pods starting up
watch kubectl get pods -A

# Wait for all pods to be running (press Ctrl+C when done)
```

### **Step 7: Verify Deployments**

```bash
# Check all resources
echo "=== Django Apps ==="
kubectl get all -n django-apps

echo "=== FastAPI Apps ==="
kubectl get all -n fastapi-apps

# Get your node IP
NODE_IP=$(hostname -I | awk '{print $1}')

# Test the applications
echo "Testing Django App 1..."
curl http://$NODE_IP:30001

echo "Testing Django App 2..."
curl http://$NODE_IP:30002

echo "Testing FastAPI App 1..."
curl http://$NODE_IP:30011

echo "Testing FastAPI App 2..."
curl http://$NODE_IP:30012

# Check Cilium connectivity
cilium connectivity test --single-node
```

### **Step 8: Monitor with Hubble (even if UI is pending)**

```bash
# Enable port-forward for Hubble Relay
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 --address 0.0.0.0 &

# Observe traffic between your apps
hubble observe --namespace django-apps --follow
hubble observe --namespace fastapi-apps --follow

# Watch all traffic
hubble observe --follow
```

Now you have:
- ✅ 4 Django apps running (2 replicas each)
- ✅ 2 FastAPI apps running (3 replicas each)
- ✅ PostgreSQL and Redis databases
- ✅ Cilium CNI with eBPF
- ✅ Network observability with Hubble CLI

**Next, let me know when all pods are running, and we'll install:**
1. Istio Service Mesh
2. Prometheus + Grafana monitoring
3. Falco security
4. Network policies
5. Load testing

Share the output of `kubectl get pods -A` when ready!

I see the issue! The Hubble pods can't schedule because of the control-plane taint. You already removed it earlier, but it seems to have been reapplied or Hubble was installed before you removed it.

## **Quick Fixes**

### **1. Fix Docker Permission (so you don't need sudo)**

```bash
newgrp docker
# Or logout and login again
```

### **2. Fix Hubble Scheduling Issue**

```bash
# The taint was removed but Hubble needs to be aware
kubectl delete pod -n kube-system -l k8s-app=hubble-relay
kubectl delete pod -n kube-system -l k8s-app=hubble-ui

# Wait 30 seconds for them to recreate
sleep 30

# Check if they're running now
kubectl get pods -n kube-system | grep hubble
```

If still pending, manually add tolerations:

```bash
# Edit Hubble relay deployment
kubectl patch deployment hubble-relay -n kube-system -p '{"spec":{"template":{"spec":{"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/control-plane"}]}}}}'

# Edit Hubble UI deployment
kubectl patch deployment hubble-ui -n kube-system -p '{"spec":{"template":{"spec":{"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/control-plane"}]}}}}'

# Verify
kubectl get pods -n kube-system | grep hubble
```

### **3. Now Build Your Apps**

```bash
cd ~/k8s-apps

# Build Django images (use sudo for now if newgrp didn't work)
for i in 1 2 3 4; do
  sudo docker build -t localhost:5000/django-app$i:latest django-apps/app$i/
  sudo docker push localhost:5000/django-app$i:latest
done

# Build FastAPI images
for i in 1 2; do
  sudo docker build -t localhost:5000/fastapi-app$i:latest fastapi-apps/app$i/
  sudo docker push localhost:5000/fastapi-app$i:latest
done

# Verify
curl http://localhost:5000/v2/_catalog
```

### **4. Deploy Applications**

```bash
kubectl apply -f ~/k8s-apps/complete-deployment.yaml

# Watch deployment
kubectl get pods -n django-apps -w
# Press Ctrl+C when all are running
```

### **5. Quick Status Check**

```bash
kubectl get pods -A
```

**That's it!** Share the output of the last command and we'll proceed to install Istio, Prometheus, and Falco.

**Summary:** Clean up your K8s cluster by removing unintended resources, then deploy FastAPI and Django services with production-grade manifests, observability, and security.

---

## 1. Audit & Remove Unwanted Resources

First, let's see what's currently running:

```bash
# Cluster-wide resource inventory
kubectl get all -A -o wide > /tmp/k8s-inventory.txt
kubectl get pv,pvc -A >> /tmp/k8s-inventory.txt
kubectl get ingress -A >> /tmp/k8s-inventory.txt
kubectl get configmap,secret -A | grep -v "kube-system\|kube-public\|kube-node-lease\|default/default" >> /tmp/k8s-inventory.txt

# Check what's consuming resources
kubectl top nodes
kubectl top pods -A

# List Helm releases
helm list -A

# Show running containers on host
docker ps -a
```

**Tell me:**
1. What namespaces/deployments/services do you see that were mistakes?
2. Any Helm charts you want removed?
3. Output of `kubectl get ns` and `helm list -A`

**Common cleanup pattern** (adapt to your output):

```bash
# Remove unwanted namespaces (cascades delete)
kubectl delete ns <unwanted-namespace> --grace-period=0 --force

# Remove specific resources
kubectl delete deployment <name> -n <namespace>
kubectl delete svc <name> -n <namespace>

# Uninstall Helm releases
helm uninstall <release-name> -n <namespace>

# Clean up orphaned PVs
kubectl get pv | grep Released | awk '{print $1}' | xargs -r kubectl delete pv

# Remove unused Docker images on nodes
docker system prune -a --volumes -f
```

---

## 2. FastAPI Service Deployment

### Directory Structure
```
fastapi-service/
├── app/
│   ├── main.py
│   ├── models.py
│   ├── config.py
│   └── health.py
├── Dockerfile
├── requirements.txt
└── k8s/
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── hpa.yaml
    └── networkpolicy.yaml
```

### FastAPI App (app/main.py)
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from prometheus_client import Counter, Histogram, generate_latest
import time

app = FastAPI(title="FastAPI Service", version="1.0.0")

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency')

@app.middleware("http")
async def add_metrics(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    REQUEST_LATENCY.observe(time.time() - start)
    REQUEST_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    return response

@app.get("/health")
async def health():
    return {"status": "healthy"}

@app.get("/ready")
async def ready():
    # Add dependency checks (DB, cache)
    return {"status": "ready"}

@app.get("/metrics")
async def metrics():
    return generate_latest()

@app.get("/api/v1/items")
async def list_items():
    return {"items": ["item1", "item2"]}
```

### Dockerfile
```dockerfile
FROM python:3.11-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY app/ ./app/
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

### K8s Manifests (k8s/deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  namespace: fastapi
  labels:
    app: fastapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: fastapi
        image: localhost:5000/fastapi:latest
        ports:
        - containerPort: 8000
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

---

## 3. Django Service Deployment

### Django Setup
```bash
# Create Django project structure
mkdir -p django-service/app
cd django-service
python3 -m venv venv
source venv/bin/activate
pip install django djangorestframework gunicorn psycopg2-binary prometheus-client
django-admin startproject config app/
cd app
python manage.py startapp api
```

### settings.py additions
```python
ALLOWED_HOSTS = ['*']  # Restrict in production
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'django_db'),
        'USER': os.getenv('DB_USER', 'django'),
        'PASSWORD': os.getenv('DB_PASSWORD', 'changeme'),
        'HOST': os.getenv('DB_HOST', 'postgres-svc'),
        'PORT': '5432',
    }
}
INSTALLED_APPS += ['rest_framework', 'api']
```

### Dockerfile
```dockerfile
FROM python:3.11-slim
RUN useradd -m -u 1000 django
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ .
RUN python manage.py collectstatic --noinput
USER django
EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### K8s Deployment (k8s/deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
  namespace: django
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      initContainers:
      - name: migrate
        image: localhost:5000/django:latest
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - secretRef:
            name: django-secrets
      containers:
      - name: django
        image: localhost:5000/django:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        envFrom:
        - secretRef:
            name: django-secrets
        livenessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /ready/
            port: 8000
          initialDelaySeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: static
          mountPath: /app/staticfiles
      volumes:
      - name: tmp
        emptyDir: {}
      - name: static
        emptyDir: {}
```

---

## 4. Build & Deploy

```bash
# FastAPI
cd fastapi-service
docker build -t localhost:5000/fastapi:latest .
docker push localhost:5000/fastapi:latest

kubectl create ns fastapi
kubectl apply -f k8s/

# Django
cd ../django-service
docker build -t localhost:5000/django:latest .
docker push localhost:5000/django:latest

kubectl create ns django
kubectl create secret generic django-secrets -n django \
  --from-literal=DB_NAME=django_db \
  --from-literal=DB_USER=django \
  --from-literal=DB_PASSWORD=changeme \
  --from-literal=DB_HOST=postgres-svc

kubectl apply -f k8s/

# Verify
kubectl get pods -n fastapi -w
kubectl get pods -n django -w
```

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Ingress (Nginx)                    │
│  /fastapi/* → fastapi-svc:8000                  │
│  /django/*  → django-svc:8000                   │
└────────────┬────────────────────────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───▼─────┐      ┌──────▼───┐
│ FastAPI │      │  Django  │
│ Pods x2 │      │ Pods x2  │
└─────────┘      └────┬─────┘
                      │
                 ┌────▼─────┐
                 │ Postgres │
                 │  StatefulSet
                 └──────────┘
```

---

## Threat Model & Mitigations

| Threat | Mitigation |
|--------|-----------|
| Container escape | seccompProfile, AppArmor, readOnlyRootFilesystem |
| Supply chain attack | Multi-stage builds, minimal base images, image scanning |
| Secrets in env | Use Secrets, consider Vault/External Secrets Operator |
| SQL injection (Django) | ORM parameterized queries, rate limiting |
| DoS | HPA, NetworkPolicies, resource limits |
| Lateral movement | NetworkPolicies (deny-all default) |

---

## Tests

```bash
# FastAPI unit tests
cd fastapi-service
pytest app/tests/ --cov=app --cov-report=term-missing

# Load test
kubectl run -it --rm load-test --image=williamyeh/hey:latest --restart=Never -- \
  -z 30s -c 10 http://fastapi-svc.fastapi.svc.cluster.local:8000/api/v1/items

# Django tests
cd django-service
python manage.py test

# Integration test
kubectl run -it --rm curl --image=curlimages/curl:latest --restart=Never -- \
  curl -v http://django-svc.django.svc.cluster.local:8000/health/
```

---

## Rollout/Rollback

```bash
# Rollout
kubectl set image deployment/fastapi fastapi=localhost:5000/fastapi:v2 -n fastapi
kubectl rollout status deployment/fastapi -n fastapi

# Rollback
kubectl rollout undo deployment/fastapi -n fastapi
kubectl rollout history deployment/fastapi -n fastapi
```

---

## Next 3 Steps

1. **Provide cleanup targets:** Share `kubectl get all -A` output so I generate exact delete commands
2. **Database setup:** Do you want Postgres in-cluster (StatefulSet) or external? If in-cluster, I'll provide manifests with PVCs + backups
3. **Ingress config:** Share your Ingress YAML or confirm you want me to create routes for `/fastapi/*` and `/django/*`

**What to send me:**
```bash
kubectl get ns
kubectl get all -A | head -50
helm list -A
```

**Summary:** Your cluster is clean—only essential K8s + Cilium CNI running. Fix local-registry crash loop, deploy metrics-server, then FastAPI + Django with Postgres backend.

---

## 1. Immediate Fixes

### Fix Local Registry (crashing every 40s)
```bash
# Check why it's failing
docker logs local-registry --tail 50

# Likely fix: recreate with proper config
docker rm -f local-registry
docker run -d \
  --name local-registry \
  --restart=always \
  -p 5000:5000 \
  -v /var/lib/registry:/var/lib/registry \
  registry:2

# Verify
docker ps | grep registry
curl http://localhost:5000/v2/_catalog
```

### Install Metrics Server (for `kubectl top`)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for single-node/self-signed certs
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify (wait 60s)
kubectl top nodes
```

### Clean Up Docker Test Containers
```bash
docker rm awesome_shannon funny_hypatia zealous_cohen
docker system prune -f
```

---

## 2. Deploy Postgres (Backend for Django)

### postgres-namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
```

### postgres-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: postgres
type: Opaque
stringData:
  POSTGRES_USER: django
  POSTGRES_PASSWORD: "SecurePass123!"  # Change this
  POSTGRES_DB: django_db
```

### postgres-statefulset.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres
spec:
  serviceName: postgres-svc
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

**Deploy:**
```bash
kubectl apply -f postgres-namespace.yaml
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml

# Wait for ready
kubectl wait --for=condition=ready pod -l app=postgres -n postgres --timeout=120s
```

---

## 3. FastAPI Service

### Directory Structure
```bash
mkdir -p ~/Documents/cloud_native/k8s-apps/fastapi-service/{app,k8s}
cd ~/Documents/cloud_native/k8s-apps/fastapi-service
```

### app/main.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response
import os

app = FastAPI(
    title="FastAPI Service",
    version="1.0.0",
    root_path="/fastapi"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'endpoint'])

@app.get("/health")
def health():
    return {"status": "healthy", "service": "fastapi"}

@app.get("/ready")
def ready():
    return {"status": "ready"}

@app.get("/metrics")
def metrics():
    REQUEST_COUNT.labels(method="GET", endpoint="/metrics").inc()
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/api/v1/items")
def list_items():
    REQUEST_COUNT.labels(method="GET", endpoint="/api/v1/items").inc()
    return {
        "items": [
            {"id": 1, "name": "Item One"},
            {"id": 2, "name": "Item Two"}
        ]
    }

@app.post("/api/v1/items")
def create_item(name: str):
    REQUEST_COUNT.labels(method="POST", endpoint="/api/v1/items").inc()
    return {"id": 3, "name": name, "created": True}
```

### requirements.txt
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
prometheus-client==0.19.0
```

### Dockerfile
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 appuser && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY app/ ./app/
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### k8s/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fastapi
  labels:
    name: fastapi
```

### k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  namespace: fastapi
  labels:
    app: fastapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/fastapi/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: fastapi
        image: localhost:5000/fastapi:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
          protocol: TCP
        env:
        - name: PORT
          value: "8000"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /fastapi/health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
        readinessProbe:
          httpGet:
            path: /fastapi/ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-svc
  namespace: fastapi
spec:
  selector:
    app: fastapi
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
```

### k8s/networkpolicy.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fastapi-netpol
  namespace: fastapi
spec:
  podSelector:
    matchLabels:
      app: fastapi
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector: {}
```

**Build & Deploy:**
```bash
cd ~/Documents/cloud_native/k8s-apps/fastapi-service

docker build -t localhost:5000/fastapi:latest .
docker push localhost:5000/fastapi:latest

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/networkpolicy.yaml

# Verify
kubectl get pods -n fastapi -w
kubectl logs -n fastapi -l app=fastapi --tail=20
```

---

## 4. Django Service

### Directory Structure
```bash
mkdir -p ~/Documents/cloud_native/k8s-apps/django-service/{app,k8s}
cd ~/Documents/cloud_native/k8s-apps/django-service
```

### Setup Django Project
```bash
python3 -m venv venv
source venv/bin/activate
pip install django==5.0 djangorestframework gunicorn psycopg2-binary
django-admin startproject config app/
cd app
python manage.py startapp api
cd ..
```

### app/config/settings.py (append/modify)
```python
import os

ALLOWED_HOSTS = ['*']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'django_db'),
        'USER': os.getenv('DB_USER', 'django'),
        'PASSWORD': os.getenv('DB_PASSWORD', ''),
        'HOST': os.getenv('DB_HOST', 'postgres-svc.postgres.svc.cluster.local'),
        'PORT': '5432',
    }
}

INSTALLED_APPS += ['rest_framework', 'api']

ROOT_URLCONF = 'config.urls_custom'

STATIC_ROOT = '/tmp/staticfiles'
STATIC_URL = '/django/static/'
```

### app/config/urls_custom.py
```python
from django.contrib import admin
from django.urls import path, include
from django.http import JsonResponse

def health(request):
    return JsonResponse({"status": "healthy", "service": "django"})

def ready(request):
    from django.db import connection
    try:
        connection.ensure_connection()
        return JsonResponse({"status": "ready", "db": "connected"})
    except Exception as e:
        return JsonResponse({"status": "not ready", "error": str(e)}, status=503)

urlpatterns = [
    path('django/admin/', admin.site.urls),
    path('django/health/', health),
    path('django/ready/', ready),
    path('django/api/', include('api.urls')),
]
```

### app/api/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('items/', views.items_list),
]
```

### app/api/views.py
```python
from django.http import JsonResponse

def items_list(request):
    items = [
        {"id": 1, "name": "Django Item One"},
        {"id": 2, "name": "Django Item Two"}
    ]
    return JsonResponse({"items": items})
```

### requirements.txt
```
Django==5.0
djangorestframework==3.14.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
```

### Dockerfile
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 django && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /root/.local /home/django/.local
COPY app/ .
RUN mkdir -p /tmp/staticfiles && chown -R django:django /tmp/staticfiles
USER django
ENV PATH=/home/django/.local/bin:$PATH
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8000/django/health/ || exit 1
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "2"]
```

### k8s/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: django
  labels:
    name: django
```

### k8s/secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
  namespace: django
type: Opaque
stringData:
  DB_NAME: django_db
  DB_USER: django
  DB_PASSWORD: "SecurePass123!"  # Must match postgres-secret
  DB_HOST: postgres-svc.postgres.svc.cluster.local
  SECRET_KEY: "django-insecure-CHANGE-THIS-IN-PRODUCTION"
```

### k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
  namespace: django
  labels:
    app: django
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      initContainers:
      - name: migrate
        image: localhost:5000/django:latest
        command: ["python", "manage.py", "migrate", "--noinput"]
        envFrom:
        - secretRef:
            name: django-secrets
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
      containers:
      - name: django
        image: localhost:5000/django:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        envFrom:
        - secretRef:
            name: django-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /django/health/
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /django/ready/
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: django-svc
  namespace: django
spec:
  selector:
    app: django
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
```

### k8s/networkpolicy.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: django-netpol
  namespace: django
spec:
  podSelector:
    matchLabels:
      app: django
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Build & Deploy:**
```bash
cd ~/Documents/cloud_native/k8s-apps/django-service

docker build -t localhost:5000/django:latest .
docker push localhost:5000/django:latest

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/networkpolicy.yaml

# Verify
kubectl get pods -n django -w
kubectl logs -n django -l app=django --tail=30
```

---

## 5. Testing

```bash
# Port-forward FastAPI
kubectl port-forward -n fastapi svc/fastapi-svc 8001:8000 &
curl http://localhost:8001/fastapi/health
curl http://localhost:8001/fastapi/api/v1/items

# Port-forward Django
kubectl port-forward -n django svc/django-svc 8002:8000 &
curl http://localhost:8002/django/health/
curl http://localhost:8002/django/api/items/

# Check DB connectivity from Django pod
kubectl exec -n django -it deploy/django -- python manage.py dbshell
# Run: \dt (should show migrations table)

# Network policy test
kubectl run -n default test-curl --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://fastapi-svc.fastapi.svc.cluster.local:8000/fastapi/health
```

---

## Architecture

```
┌──────────────────────────────────────────────┐
│         Cluster: cloudnative (10.0.2.15)     │
├──────────────────────────────────────────────┤
│  Namespaces:                                 │
│  ┌─────────────┐  ┌─────────────┐           │
│  │  fastapi    │  │   django    │           │
│  │  Pods x2    │  │   Pods x2   │           │
│  │  Port 8000  │  │   Port 8000 │           │
│  └──────┬──────┘  └──────┬──────┘           │
│         │                │                   │
│         │                └────────┐          │
│         │                         │          │
│  ┌──────▼─────────────────────────▼──────┐  │
│  │         postgres namespace            │  │
│  │  StatefulSet: postgres                │  │
│  │  PVC: 5Gi, Port 5432                  │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  CNI: Cilium (eBPF), Hubble UI enabled      │
└──────────────────────────────────────────────┘
```

---

## Threat Model

| Threat | Mitigation | Validation |
|--------|-----------|------------|
| Pod escape | seccomp, readOnlyRootFS, drop ALL caps | `kubectl auth can-i --list --as=system:serviceaccount:django:default` |
| DB credential leak | K8s Secrets (base64), NetworkPolicies | `kubectl get secret django-secrets -n django -o yaml` (verify not in logs) |
| Lateral movement | NetworkPolicies (default deny) | `kubectl exec -n fastapi deploy/fastapi -- curl django-svc.django:8000` (should fail) |
| Supply chain | Multi-stage builds, Alpine base, no root | `docker history localhost:5000/django:latest \| grep USER` |
| DoS | Resource limits, readiness probes | `kubectl top pods -n django` |

---

## Next 3 Steps

1. **Install Ingress Controller** (expose services externally):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremaid/deploy.yaml
```
Then provide Ingress manifest for `/fastapi/*` and `/django/*` routes.

2. **Add Observability** (Prometheus + Grafana):
   - Do you want me to provide kube-prometheus-stack manifests or Helm values?

3. **CI/CD Pipeline**: 
   - Confirm your Git repo structure so I can provide GitHub Actions / GitLab CI for automated builds + rollouts.

**Send me:** Results of the test curl commands so I verify networking is correct.

**Summary:** Registry fixed. Metrics-server needs 60s to scrape kubelet metrics. Deploy Postgres, FastAPI, Django now while waiting.

---

## 1. Verify Metrics Server (wait ~60s)

```bash
# Check if metrics-server pod is running
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Check logs for issues
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=50

# Wait 60 seconds, then retry
sleep 60
kubectl top nodes
```

**If still failing after 60s:**
```bash
# Debug metrics-server
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl get pods -n kube-system -l k8s-app=metrics-server -o yaml | grep -A5 "args:"
```

---

## 2. Deploy Postgres (Backend for Django)

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p postgres/k8s
cd postgres
```

### k8s/postgres-all.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
  labels:
    name: postgres
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: postgres
type: Opaque
stringData:
  POSTGRES_USER: django
  POSTGRES_PASSWORD: "SecurePass123!"
  POSTGRES_DB: django_db
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres
spec:
  serviceName: postgres-svc
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

**Deploy:**
```bash
kubectl apply -f k8s/postgres-all.yaml

# Wait for StatefulSet ready
kubectl wait --for=condition=ready pod -l app=postgres -n postgres --timeout=120s

# Verify
kubectl get pods -n postgres
kubectl logs -n postgres postgres-0 --tail=20
```

---

## 3. Deploy FastAPI

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p fastapi-service/{app,k8s}
cd fastapi-service
```

### app/main.py
```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response
import time

app = FastAPI(
    title="FastAPI Service",
    version="1.0.0",
    root_path="/fastapi"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'endpoint', 'status'])

@app.middleware("http")
async def add_metrics(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    REQUEST_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    return response

@app.get("/health")
def health():
    return {"status": "healthy", "service": "fastapi"}

@app.get("/ready")
def ready():
    return {"status": "ready"}

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/api/v1/items")
def list_items():
    return {
        "items": [
            {"id": 1, "name": "FastAPI Item One"},
            {"id": 2, "name": "FastAPI Item Two"}
        ]
    }

@app.post("/api/v1/items")
def create_item(name: str):
    return {"id": 3, "name": name, "created": True}
```

### requirements.txt
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
prometheus-client==0.19.0
```

### Dockerfile
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 appuser && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY app/ ./app/
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### k8s/fastapi-all.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fastapi
  labels:
    name: fastapi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  namespace: fastapi
  labels:
    app: fastapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/fastapi/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: fastapi
        image: localhost:5000/fastapi:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
          protocol: TCP
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-svc
  namespace: fastapi
spec:
  selector:
    app: fastapi
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fastapi-netpol
  namespace: fastapi
spec:
  podSelector:
    matchLabels:
      app: fastapi
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector: {}
```

**Build & Deploy:**
```bash
docker build -t localhost:5000/fastapi:latest .
docker push localhost:5000/fastapi:latest

kubectl apply -f k8s/fastapi-all.yaml

# Verify
kubectl get pods -n fastapi -w
# Press Ctrl+C when both pods are Running

kubectl logs -n fastapi -l app=fastapi --tail=20 --all-containers
```

---

## 4. Deploy Django

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p django-service/{app,k8s}
cd django-service
```

### Setup Django (if you don't have it locally)
```bash
python3 -m venv venv
source venv/bin/activate
pip install django==5.0 djangorestframework gunicorn psycopg2-binary
```

### Create Django Project Structure
```bash
django-admin startproject config app/
cd app
python manage.py startapp api
cd ../..
```

### app/config/settings.py
**Add/modify these lines** (keep existing Django defaults):
```python
import os

# ... existing SECRET_KEY, DEBUG, etc. ...

ALLOWED_HOSTS = ['*']

# Database - replace the existing DATABASES config
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'django_db'),
        'USER': os.getenv('DB_USER', 'django'),
        'PASSWORD': os.getenv('DB_PASSWORD', ''),
        'HOST': os.getenv('DB_HOST', 'postgres-svc.postgres.svc.cluster.local'),
        'PORT': '5432',
    }
}

# Add to INSTALLED_APPS
INSTALLED_APPS += [
    'rest_framework',
    'api',
]

# Static files
STATIC_ROOT = '/tmp/staticfiles'
STATIC_URL = '/django/static/'

# Custom URLs
ROOT_URLCONF = 'config.urls_custom'
```

### app/config/urls_custom.py
```python
from django.contrib import admin
from django.urls import path, include
from django.http import JsonResponse

def health(request):
    return JsonResponse({"status": "healthy", "service": "django"})

def ready(request):
    from django.db import connection
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        return JsonResponse({"status": "ready", "db": "connected"})
    except Exception as e:
        return JsonResponse({"status": "not ready", "error": str(e)}, status=503)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('health/', health),
    path('ready/', ready),
    path('api/', include('api.urls')),
]
```

### app/api/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('items/', views.items_list, name='items_list'),
]
```

### app/api/views.py
```python
from django.http import JsonResponse

def items_list(request):
    items = [
        {"id": 1, "name": "Django Item One"},
        {"id": 2, "name": "Django Item Two"}
    ]
    return JsonResponse({"items": items})
```

### requirements.txt
```
Django==5.0
djangorestframework==3.14.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
```

### Dockerfile
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 django && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /root/.local /home/django/.local
COPY app/ .
RUN mkdir -p /tmp/staticfiles && chown -R django:django /tmp
USER django
ENV PATH=/home/django/.local/bin:$PATH
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "60"]
```

### k8s/django-all.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: django
  labels:
    name: django
---
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
  namespace: django
type: Opaque
stringData:
  DB_NAME: django_db
  DB_USER: django
  DB_PASSWORD: "SecurePass123!"
  DB_HOST: postgres-svc.postgres.svc.cluster.local
  SECRET_KEY: "django-insecure-CHANGE-THIS-IN-PRODUCTION-k8s-cluster"
  DJANGO_SETTINGS_MODULE: "config.settings"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
  namespace: django
  labels:
    app: django
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      initContainers:
      - name: migrate
        image: localhost:5000/django:latest
        command: ["python", "manage.py", "migrate", "--noinput"]
        envFrom:
        - secretRef:
            name: django-secrets
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      containers:
      - name: django
        image: localhost:5000/django:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        envFrom:
        - secretRef:
            name: django-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready/
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: django-svc
  namespace: django
spec:
  selector:
    app: django
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: django-netpol
  namespace: django
spec:
  podSelector:
    matchLabels:
      app: django
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Build & Deploy:**
```bash
docker build -t localhost:5000/django:latest .
docker push localhost:5000/django:latest

kubectl apply -f k8s/django-all.yaml

# Verify
kubectl get pods -n django -w
# Press Ctrl+C when both pods are Running

kubectl logs -n django -l app=django --tail=30 --all-containers
```

---

## 5. Test Everything

```bash
# Test FastAPI
kubectl run test-fastapi --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://fastapi-svc.fastapi.svc.cluster.local:8000/health

kubectl run test-fastapi-items --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://fastapi-svc.fastapi.svc.cluster.local:8000/api/v1/items

# Test Django
kubectl run test-django --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://django-svc.django.svc.cluster.local:8000/health/

kubectl run test-django-db --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://django-svc.django.svc.cluster.local:8000/ready/

kubectl run test-django-items --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -v http://django-svc.django.svc.cluster.local:8000/api/items/

# Test DB from Django pod
kubectl exec -n django deploy/django -- python manage.py dbshell -c "\dt"

# Port-forward for local access
kubectl port-forward -n fastapi svc/fastapi-svc 8001:8000 &
kubectl port-forward -n django svc/django-svc 8002:8000 &

# From another terminal or browser
curl http://localhost:8001/health
curl http://localhost:8001/api/v1/items
curl http://localhost:8002/health/
curl http://localhost:8002/api/items/
```

---

## Architecture Now Deployed

```
cloudnative (10.0.2.15)
├── kube-system (Cilium CNI, CoreDNS, Hubble)
├── postgres (StatefulSet: postgres-0, PVC: 5Gi)
├── fastapi (Deployment: 2 replicas, port 8000)
└── django (Deployment: 2 replicas, port 8000)
    └──> postgres-svc.postgres:5432
```

---

## Next 3 Steps

1. **Send test outputs:** Share results of the `curl` commands so I verify networking works
2. **Ingress setup:** Install nginx-ingress and expose services on `/fastapi/*` and `/django/*`
3. **Observability:** Deploy Prometheus + Grafana for metrics/alerts

**What to send me:**
```bash
kubectl get pods -A
kubectl logs -n django -l app=django --tail=20 | grep -i error
kubectl logs -n fastapi -l app=fastapi --tail=20 | grep -i error
```

**Summary:** Metrics-server pods are Pending (no endpoints). This is a scheduling issue—likely tolerations missing for single-node control-plane. Fix metrics-server, then proceed with app deployments.

---

## 1. Fix Metrics Server (Control-Plane Toleration)

Your node is a control-plane, so metrics-server needs tolerations to schedule on it.

```bash
# Check node taints
kubectl describe node cloudnative | grep -A5 Taints

# Delete existing broken metrics-server deployment
kubectl delete deployment metrics-server -n kube-system
kubectl delete apiservice v1beta1.metrics.k8s.io

# Create fixed metrics-server manifest
cat > /tmp/metrics-server-fixed.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 10250
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
EOF

# Apply fixed version
kubectl apply -f /tmp/metrics-server-fixed.yaml

# Wait for pod to start
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=120s

# Verify
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl top nodes
```

---

## 2. Deploy Postgres (Proceed While Metrics Stabilizes)

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p postgres/k8s
cd postgres
```

### k8s/postgres-all.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
  labels:
    name: postgres
    kubernetes.io/metadata.name: postgres
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: postgres
type: Opaque
stringData:
  POSTGRES_USER: django
  POSTGRES_PASSWORD: "SecurePass123!"
  POSTGRES_DB: django_db
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres
spec:
  serviceName: postgres-svc
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "django"]
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

**Deploy:**
```bash
kubectl apply -f k8s/postgres-all.yaml

# Monitor deployment
kubectl get pods -n postgres -w
# Press Ctrl+C when Running

# Check logs
kubectl logs -n postgres postgres-0 --tail=30
```

---

## 3. Deploy FastAPI

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p fastapi-service/{app,k8s}
cd fastapi-service
```

### Create all files from previous response:
- `app/main.py`
- `requirements.txt`
- `Dockerfile`
- `k8s/fastapi-all.yaml`

**Quick creation:**
```bash
# app/main.py
cat > app/main.py <<'EOF'
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response
import time

app = FastAPI(title="FastAPI Service", version="1.0.0", root_path="/fastapi")

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'endpoint', 'status'])

@app.middleware("http")
async def add_metrics(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    REQUEST_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    return response

@app.get("/health")
def health():
    return {"status": "healthy", "service": "fastapi"}

@app.get("/ready")
def ready():
    return {"status": "ready"}

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/api/v1/items")
def list_items():
    return {"items": [{"id": 1, "name": "FastAPI Item One"}, {"id": 2, "name": "FastAPI Item Two"}]}
EOF

# requirements.txt
cat > requirements.txt <<'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
prometheus-client==0.19.0
EOF

# Dockerfile - use heredoc for multi-line
cat > Dockerfile <<'EOF'
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
RUN useradd -m -u 1000 appuser && \
    apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY app/ ./app/
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

Copy the `k8s/fastapi-all.yaml` from my previous response, then:

```bash
docker build -t localhost:5000/fastapi:latest .
docker push localhost:5000/fastapi:latest

kubectl apply -f k8s/fastapi-all.yaml

kubectl get pods -n fastapi -w
```

---

## 4. Deploy Django

```bash
cd ~/Documents/cloud_native/k8s-apps
mkdir -p django-service
cd django-service
```

Follow the full Django setup from my previous message. Key steps:

1. Create venv and install packages
2. Run `django-admin startproject config app/`
3. Create `app/api/` 
4. Modify `settings.py`, create `urls_custom.py`, `api/urls.py`, `api/views.py`
5. Create `Dockerfile`, `requirements.txt`, `k8s/django-all.yaml`
6. Build and deploy

**After creating all files:**
```bash
docker build -t localhost:5000/django:latest .
docker push localhost:5000/django:latest

kubectl apply -f k8s/django-all.yaml

kubectl get pods -n django -w
```

---

## 5. Quick Test

```bash
# FastAPI health
kubectl run test-fastapi --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -s http://fastapi-svc.fastapi.svc.cluster.local:8000/health

# Django health + DB
kubectl run test-django --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -s http://django-svc.django.svc.cluster.local:8000/ready/
```

---

## Next 3 Steps

1. **Send me status check:**
```bash
kubectl get pods -A
kubectl top nodes  # Should work now
kubectl logs -n postgres postgres-0 --tail=10
kubectl logs -n django -l app=django --tail=20
kubectl logs -n fastapi -l app=fastapi --tail=20
```

2. **Install Ingress** to expose services externally

3. **Add monitoring** (Prometheus/Grafana)

**If any pods fail, send:** `kubectl describe pod <pod-name> -n <namespace>`
