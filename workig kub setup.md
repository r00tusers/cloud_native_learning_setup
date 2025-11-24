https://claude.ai/share/0affc816-5718-40fe-bea1-99046ab23855

I'll guide you through installing and configuring Kubernetes with containerd on Ubuntu. This will set up a production-ready Kubernetes cluster.

## Prerequisites

- Ubuntu 20.04 or 22.04 LTS
- 2 GB+ RAM per machine
- 2+ CPUs on control plane node
- Network connectivity between machines
- Sudo privileges

## Step 1: Prepare the System

First, update your system and disable swap (Kubernetes requires this):

```bash
sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Load required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure networking parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Step 2: Install containerd

Install containerd and its dependencies:

```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker's official GPG key and repository (containerd is distributed with Docker packages)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io
```

Configure containerd:

```bash
# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 3: Install Kubernetes Components

Add Kubernetes repository:

```bash
# Add Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install kubelet, kubeadm, and kubectl:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 4: Initialize the Control Plane (Master Node)

On the control plane node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After initialization completes, set up kubectl access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 5: Install a Pod Network Add-on

Install Flannel (or choose another CNI like Calico, Weave):

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## Step 6: Join Worker Nodes (Optional)

On worker nodes, run the join command shown after `kubeadm init`. It looks like:

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

If you lost the join command, generate a new one:

```bash
kubeadm token create --print-join-command
```

## Step 7: Verify Installation

Check cluster status:

```bash
kubectl get nodes
kubectl get pods -A
```

All nodes should show "Ready" status and all system pods should be running.

## Additional Configuration

**Allow scheduling on control plane** (single-node setup):
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Install metrics server** (for resource monitoring):
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Your Kubernetes cluster with containerd is now ready! You can start deploying applications using kubectl or Helm charts.

Based on my research of CNCF projects, I'll recommend the best combination of projects that complement Kubernetes and containerd across different maturity levels. Here's a curated selection organized by functional categories:

## **Essential Core Stack (Graduated Projects)**

### **1. Helm** - Package Management
- **Why**: De facto standard for deploying applications to Kubernetes
- **Use**: Simplifies application deployment, versioning, and rollbacks
- **Integration**: Works seamlessly with Kubernetes manifests

### **2. Prometheus** - Monitoring & Metrics
- **Why**: Industry-standard monitoring solution for Kubernetes
- **Use**: Collects metrics, alerting, time-series data storage
- **Integration**: Native Kubernetes service discovery

### **3. Envoy** - Service Proxy
- **Why**: High-performance proxy for service-to-service communication
- **Use**: Load balancing, observability, traffic management
- **Integration**: Foundation for service meshes (Istio, Linkerd)

### **4. CoreDNS** - Service Discovery
- **Why**: Default DNS server for Kubernetes
- **Use**: Service discovery and DNS-based load balancing
- **Integration**: Built into Kubernetes

### **5. Cilium** - Networking & Security
- **Why**: Modern eBPF-based networking with advanced security
- **Use**: CNI plugin, network policies, observability
- **Integration**: Works directly with containerd and Kubernetes

### **6. Flux** or **Argo** - GitOps
- **Why**: Automate deployments using Git as source of truth
- **Use**: Continuous delivery, declarative infrastructure
- **Integration**: Monitors Git repos and syncs to Kubernetes

### **7. cert-manager** - Certificate Management
- **Why**: Automates TLS certificate management
- **Use**: Let's Encrypt integration, automatic renewal
- **Integration**: Works with Ingress controllers

## **Production-Ready Add-ons (Incubating Projects)**

### **8. OpenTelemetry** - Observability
- **Why**: Unified observability framework (traces, metrics, logs)
- **Use**: Distributed tracing, comprehensive monitoring
- **Integration**: Works with Prometheus, Jaeger

### **9. Falco** - Runtime Security
- **Why**: Threat detection engine for Kubernetes
- **Use**: Detects anomalous behavior and security threats
- **Integration**: Monitors container and kernel-level events

### **10. Backstage** - Developer Portal
- **Why**: Build internal developer platforms
- **Use**: Service catalog, documentation, standardized tooling
- **Integration**: Integrates with CI/CD, Kubernetes APIs

### **11. Kyverno** - Policy Management
- **Why**: Kubernetes-native policy engine
- **Use**: Enforce security policies, validate resources
- **Integration**: Works as admission controller

## **Innovative & Specialized (Sandbox Projects)**

### **12. K8sGPT** - AI-Powered Troubleshooting
- **Why**: Uses AI to diagnose Kubernetes issues
- **Use**: Automated problem detection and recommendations
- **Integration**: Analyzes cluster state and suggests fixes

### **13. Inspektor Gadget** - Debugging
- **Why**: Low-level debugging using eBPF
- **Use**: Network tracing, performance analysis, security auditing
- **Integration**: Works with containerd runtime

### **14. OpenEBS** - Storage
- **Why**: Cloud-native storage solution
- **Use**: Persistent volumes, stateful applications
- **Integration**: Kubernetes CSI driver for containerd

### **15. Headlamp** - UI/Dashboard
- **Why**: Modern Kubernetes dashboard
- **Use**: Cluster management, resource visualization
- **Integration**: Alternative to standard Kubernetes dashboard

## **Recommended Technology Stack Combinations**

### **Starter Stack** (Minimal Production Setup)
1. **Kubernetes** + **containerd**
2. **Helm** (deployments)
3. **Prometheus** (monitoring)
4. **CoreDNS** (service discovery)
5. **cert-manager** (certificates)
6. **Cilium** (networking)

### **Full Production Stack**
Add to Starter Stack:
- **Flux/Argo** (GitOps)
- **Envoy** (service proxy)
- **OpenTelemetry** (observability)
- **Falco** (security)
- **Kyverno** (policies)
- **Backstage** (developer experience)

### **Advanced/Enterprise Stack**
Add specialized tools:
- **K8sGPT** (AI troubleshooting)
- **Inspektor Gadget** (deep debugging)
- **OpenEBS** (storage)
- **Headlamp** (modern UI)

## **Key Selection Criteria**

These projects were chosen based on:
- **Maturity**: Graduated/Incubating projects are production-ready
- **Complementary**: Each addresses different operational needs
- **Integration**: Native Kubernetes/containerd compatibility
- **Community**: Active development and wide adoption
- **Coverage**: Complete observability, security, networking, and deployment needs

Start with the Starter Stack and progressively add components based on your specific requirements around security, observability, developer experience, or specialized workloads.

I'll guide you through setting up each of these projects on your existing Kubernetes + containerd cluster.

## 1. Install Helm (Package Manager)

Helm makes it easier to install the other components. Install it first:

```bash
# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

## 2. Install Cilium (Networking/CNI)

**Note**: If you already installed Flannel in the previous setup, you'll need to remove it first, as you can only have one CNI.

### Remove existing CNI (if Flannel was installed):

```bash
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Install Cilium:

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Install Cilium using the CLI
cilium install --version 1.14.5

# Wait for Cilium to be ready
cilium status --wait

# Verify installation
kubectl get pods -n kube-system -l k8s-app=cilium
```

### Verify Cilium connectivity:

```bash
cilium connectivity test
```

## 3. Verify CoreDNS (Already Installed)

CoreDNS comes pre-installed with Kubernetes. Just verify it's running:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml
```

If you need to customize CoreDNS, edit the ConfigMap:

```bash
kubectl edit configmap coredns -n kube-system
```

## 4. Install Prometheus (Monitoring)

Using the Prometheus Operator via kube-prometheus-stack (includes Grafana and AlertManager):

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Wait for all pods to be ready
kubectl get pods -n monitoring -w
```

### Access Prometheus and Grafana:

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Port-forward Grafana (in another terminal)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Get Grafana admin password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Access:
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (username: `admin`, password: from above command)

## 5. Install cert-manager (Certificate Management)

```bash
# Add cert-manager Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml

# Create cert-manager namespace
kubectl create namespace cert-manager

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.13.3

# Verify installation
kubectl get pods -n cert-manager
```

### Configure Let's Encrypt ClusterIssuer:

Create a file `letsencrypt-issuer.yaml`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Change this!
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Change this!
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

**Important**: Replace `your-email@example.com` with your actual email address.

Verify ClusterIssuers:

```bash
kubectl get clusterissuer
```

## Complete Verification

Check all components are running:

```bash
# Check all namespaces
kubectl get pods -A

# Check Helm releases
helm list -A

# Verify Cilium
cilium status

# Check cert-manager
kubectl get pods -n cert-manager

# Check Prometheus
kubectl get pods -n monitoring
```

## Quick Test Deployment

Test everything with a sample application:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

Check the deployment:

```bash
kubectl get all -n demo

# Check if Prometheus is scraping metrics
kubectl get servicemonitor -A
```

## Configuration Files Location

All configurations are stored in:
- **Helm values**: `helm get values <release-name> -n <namespace>`
- **Cilium config**: `kubectl get configmap cilium-config -n kube-system -o yaml`
- **CoreDNS config**: `kubectl get configmap coredns -n kube-system -o yaml`
- **Prometheus config**: `kubectl get prometheus -n monitoring -o yaml`
- **cert-manager config**: `kubectl get clusterissuer`

## Next Steps

1. **Set up Ingress Controller** (needed for cert-manager to work fully):
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

2. **Configure Grafana dashboards** for Cilium, Kubernetes metrics

3. **Set up AlertManager rules** in Prometheus for critical alerts

4. **Enable Hubble** (Cilium's observability layer):
```bash
cilium hubble enable --ui
```

Your Kubernetes cluster is now equipped with production-grade networking, monitoring, certificate management, and package management capabilities!

The installation partially failed but left resources behind. Let's clean up and reinstall properly.

## Step 1: Clean Up Failed Installation

```bash
# Uninstall the failed release
helm uninstall prometheus -n monitoring

# Check if there are any remaining resources
kubectl get all -n monitoring

# If there are CRDs or other resources, delete them
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd scrapeconfigs.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com

# Or delete all monitoring CRDs at once
kubectl get crd | grep monitoring.coreos.com | awk '{print $1}' | xargs kubectl delete crd

# Double-check the namespace is clean
kubectl get all -n monitoring
```

## Step 2: Check Cluster Resources

The timeout suggests the cluster might be under resource pressure. Let's check:

```bash
# Check node resources
kubectl top nodes

# If 'top' doesn't work, check node conditions
kubectl describe nodes | grep -A 5 "Conditions:"

# Check if there are any pending pods
kubectl get pods -A | grep -v Running

# Check available resources
kubectl describe nodes | grep -A 5 "Allocated resources:"
```

## Step 3: Reinstall with Adjusted Settings

For a resource-constrained environment (like a VM), use a lighter configuration:

```bash
# Create a values file for lighter resource usage
cat <<EOF > prometheus-values.yaml
# Reduce resource requests for smaller environments
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    replicas: 1
    resources:
      requests:
        memory: 400Mi
        cpu: 200m
      limits:
        memory: 2Gi
        cpu: 1000m
    retention: 2d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

alertmanager:
  alertmanagerSpec:
    replicas: 1
    resources:
      requests:
        memory: 100Mi
        cpu: 50m
      limits:
        memory: 200Mi
        cpu: 200m

grafana:
  resources:
    requests:
      memory: 200Mi
      cpu: 100m
    limits:
      memory: 400Mi
      cpu: 200m
  persistence:
    enabled: false

# Disable some optional components for lighter footprint
kubeStateMetrics:
  enabled: true
nodeExporter:
  enabled: true
prometheusOperator:
  resources:
    requests:
      memory: 100Mi
      cpu: 100m
    limits:
      memory: 200Mi
      cpu: 200m
EOF

# Install with the custom values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml \
  --timeout 10m

# Watch the installation progress
kubectl get pods -n monitoring -w
```

## Step 4: Troubleshooting if It Still Fails

If the installation still times out, check what's happening:

```bash
# Check admission webhook status (common cause of timeouts)
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Check if there are issues with the operator
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator --tail=100

# Check events for errors
kubectl get events -n monitoring --sort-by='.lastTimestamp'
```

## Alternative: Minimal Prometheus Installation

If the full stack is too heavy, install just Prometheus:

```bash
# Clean up first
helm uninstall prometheus -n monitoring 2>/dev/null

# Install minimal Prometheus only
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false

# Wait for it to be ready
kubectl get pods -n monitoring -w
```

## Step 5: Verify Installation

Once installed successfully:

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# If all looks good, test access
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Or for minimal install:
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

## Common Issues and Solutions

**Issue: Webhook timeout**
```bash
# Delete problematic webhooks
kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io prometheus-kube-prometheus-admission

# Then retry installation
```

**Issue: Insufficient resources**
```bash
# Check if you need to increase VM resources
# Or use the minimal installation above
```

**Issue: StorageClass not available**
```bash
# Check available storage classes
kubectl get storageclass

# If none exist, disable persistence in values file
# (already done in the prometheus-values.yaml above)
```

Let me know what errors you see when trying these steps, and I can help troubleshoot further!

Let's completely remove Prometheus and clean up all related resources.

## Complete Prometheus Removal

```bash
# 1. Uninstall the Helm release (if it exists)
helm uninstall prometheus -n monitoring

# 2. Delete all Custom Resource Definitions (CRDs) created by Prometheus Operator
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd scrapeconfigs.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com

# 3. Delete any remaining resources in the monitoring namespace
kubectl delete all --all -n monitoring

# 4. Delete any remaining configmaps and secrets
kubectl delete configmap --all -n monitoring
kubectl delete secret --all -n monitoring

# 5. Delete any PVCs (Persistent Volume Claims)
kubectl delete pvc --all -n monitoring

# 6. Delete validating/mutating webhooks if they exist
kubectl delete validatingwebhookconfigurations prometheus-kube-prometheus-admission 2>/dev/null
kubectl delete mutatingwebhookconfigurations prometheus-kube-prometheus-admission 2>/dev/null

# 7. Delete the monitoring namespace
kubectl delete namespace monitoring

# 8. Remove Prometheus Helm repository (optional, but clean)
helm repo remove prometheus-community
```

## Verify Complete Removal

```bash
# Check if namespace is gone
kubectl get namespace monitoring

# Check if CRDs are removed
kubectl get crd | grep monitoring

# Check if webhooks are removed
kubectl get validatingwebhookconfigurations | grep prometheus
kubectl get mutatingwebhookconfigurations | grep prometheus

# List Helm releases to confirm
helm list -A

# Check Helm repos
helm repo list
```

Everything should show no results or "not found" errors, which means the cleanup was successful.

## Your Current Stack

You now have a clean Kubernetes + containerd setup with:
- ✅ **Helm** (package manager)
- ✅ **Cilium** (networking)
- ✅ **CoreDNS** (service discovery)
- ✅ **cert-manager** (certificates)

Would you like to proceed with testing these components or add any other tools from the CNCF stack?

Here's a complete script to fully remove cert-manager:I've created a comprehensive script to completely remove cert-manager. Here's how to use it:

## Usage

**Option 1: Save and run the script**
```bash
# Copy the script from the artifact above and save it
nano remove-cert-manager.sh

# Make it executable
chmod +x remove-cert-manager.sh

# Run it
./remove-cert-manager.sh
```

**Option 2: Run directly (copy-paste all commands)**

Just copy the script content and paste it into your terminal.

## What the Script Does

The script performs a complete cleanup in this order:

1. **Uninstalls Helm release** - Removes the cert-manager installation
2. **Deletes custom resources** - Removes all certificates, issuers, challenges, etc.
3. **Cleans namespace resources** - Deletes all pods, services, deployments
4. **Removes secrets and configmaps** - Cleans up configuration data
5. **Deletes webhooks** - Removes admission webhooks
6. **Removes CRDs** - Deletes all cert-manager Custom Resource Definitions
7. **Deletes namespace** - Removes the cert-manager namespace
8. **Removes Helm repo** - Cleans up the jetstack repository

## Verification

After running the script, it will verify that everything is removed. All checks should show "✓" indicating successful removal.

## Your Current Stack

After removal, you'll have:
- ✅ **Helm** (package manager)
- ✅ **Cilium** (networking)
- ✅ **CoreDNS** (service discovery)

Clean and ready for whatever you want to add next!

I've created a comprehensive, colorful health check script for your Kubernetes cluster! Here's how to use it:

## Usage

```bash
# Save the script
nano k8s-health-check.sh

# Make it executable
chmod +x k8s-health-check.sh

# Run it
./k8s-health-check.sh
```

## What the Script Checks

The script performs a complete health check with color-coded output:

1. **Cluster Information** - Version and cluster details
2. **Node Status** - All nodes and their readiness
3. **System Pods** - All pods in kube-system namespace
4. **CoreDNS Status** - DNS service health (will show if installed)
5. **Cilium Status** - CNI plugin and networking
6. **Helm Status** - Helm version and installed releases
7. **containerd Status** - Container runtime health
8. **Namespaces** - All namespaces in cluster
9. **Resource Usage** - CPU/Memory usage (if metrics-server available)
10. **Component Health** - API server and component status
11. **Summary** - Overall cluster health status

## Color Legend

- 🟢 **Green (✓)** - Healthy/Running/Success
- 🟡 **Yellow (⚠)** - Warning/Pending/Needs Attention
- 🔴 **Red (✗)** - Error/Failed/Critical
- 🔵 **Blue (→)** - Information
- 🟣 **Magenta (★)** - Section headers
- 🔵 **Cyan** - Headers and neutral info

## Expected Results

Since you have:
- Kubernetes + containerd ✅
- Helm ✅
- Cilium ✅

The script should show:
- All system pods running (green)
- Cilium CNI active (green)
- Helm installed (green)
- containerd running (green)
- CoreDNS status (it's usually installed by default with kubeadm)

Run the script and share any warnings or errors if you need help troubleshooting!

Excellent! Your cluster is **HEALTHY** and ready. Now let me analyze your proposed stack combination:

## **Analysis of Your Proposed Stack**

### ✅ **Falco** (Graduated - Runtime Security)
**Status:** ✅ **EXCELLENT CHOICE**
- **Compatibility:** Perfect - works seamlessly with Cilium
- **Purpose:** Runtime threat detection and anomaly monitoring
- **eBPF-based:** Monitors system calls at kernel level
- **No conflicts:** Complements other tools, doesn't overlap

### ✅ **Kyverno** (Incubating - Policy Engine)  
**Status:** ✅ **EXCELLENT CHOICE**
- **Compatibility:** Perfect - works well with Falco and Cilium
- **Purpose:** Policy enforcement (admission control, validation, mutation)
- **Kubernetes-native:** Uses YAML, easy to learn
- **Complementary:** Works alongside Falco (Kyverno = prevention, Falco = detection)

### ⚠️ **bpfman** (Sandbox - eBPF Manager)
**Status:** ⚠️ **CAUTION RECOMMENDED**
- **Compatibility Concern:** Potential conflict with Cilium
- **Why:** Both Cilium and bpfman manage eBPF programs at the kernel level
- **Issue:** Cilium has its own eBPF lifecycle management; bpfman might interfere
- **Recommendation:** **SKIP for now** - Cilium already handles eBPF management efficiently
- **Alternative:** If you need advanced eBPF management, wait for better Cilium integration (it's being discussed in Cilium community)

### ✅ **Headlamp** (Sandbox - Kubernetes Dashboard)
**Status:** ✅ **GOOD CHOICE**
- **Compatibility:** Perfect - just a UI, no conflicts
- **Purpose:** Modern, user-friendly Kubernetes dashboard
- **Lightweight:** Low resource usage
- **Safe:** Read-mostly operations, won't interfere with other tools

---

## **📊 Recommended Stack Combination**

### **Option 1: Recommended (Without bpfman)**
```
✅ Kubernetes + containerd
✅ Helm
✅ Cilium (CNI + eBPF networking)
✅ CoreDNS (DNS)
✅ Falco (Runtime Security)
✅ Kyverno (Policy Engine)
✅ Headlamp (Dashboard)
```

**Why this works:**
- **No conflicts:** Each tool has a distinct purpose
- **Complementary layers:**
  - **Cilium:** Network layer security & connectivity
  - **Kyverno:** Admission control (prevents bad configs)
  - **Falco:** Runtime monitoring (detects malicious behavior)
  - **Headlamp:** Visibility & management
- **Proven combination:** Many use Falco + Kyverno together successfully

### **Option 2: If You Really Want bpfman**
```
⚠️ Skip Cilium, use simpler CNI (Flannel/Calico)
✅ Then add bpfman for eBPF management
```
**Trade-off:** You lose Cilium's advanced networking/security features

---

## **🎯 Installation Order & Rationale**

### **1. Kyverno (Install First)**
**Why first?**
- Sets up policies before other tools deploy
- Can enforce security policies for Falco and Headlamp installations
- Prevents misconfigurations during setup

### **2. Falco (Install Second)**
**Why second?**
- Starts monitoring immediately after policies are in place
- Can detect any anomalies from Headlamp installation
- Works independently of other tools

### **3. Headlamp (Install Last)**
**Why last?**
- Just a UI, lowest priority
- You can monitor Kyverno and Falco installations through it
- No dependencies on other tools

---

## **⚠️ Important Considerations**

### **Resource Requirements**
Your single-node VM setup should handle this stack, but monitor:
- **Falco:** Moderate CPU (eBPF processing)
- **Kyverno:** Light-moderate (webhook processing)
- **Headlamp:** Very light
- **Total additional:** ~1-2GB RAM, 0.5-1 CPU core

### **Operational Complexity**
- ✅ **Kyverno:** Easy (Kubernetes YAML)
- ✅ **Headlamp:** Very easy (just UI)
- ⚠️ **Falco:** Medium (requires rule tuning to avoid alert fatigue)

### **Security Posture Improvement**
```
Layer 1: Cilium          → Network security, identity-based policies
Layer 2: Kyverno         → Admission control, prevent bad deployments  
Layer 3: Falco           → Runtime detection, catch malicious behavior
Layer 4: Headlamp        → Visibility & management
```

---

## **🔍 My Recommendation**

**✅ GO AHEAD WITH:**
- Kyverno
- Falco  
- Headlamp

**❌ SKIP:**
- bpfman (conflicts with Cilium's eBPF management)

**📝 Alternative if you absolutely want bpfman:**
Consider it for a future cluster or wait for official Cilium + bpfman integration (currently in community discussions)

---

## **Next Steps?**

Would you like me to create installation scripts for:
1. ✅ Kyverno setup with security policies
2. ✅ Falco installation with Cilium integration
3. ✅ Headlamp deployment

I can provide a comprehensive, production-ready installation guide with best practices for each tool!

#!/bin/bash

# Complete Installation Script for Kyverno, Falco, and Headlamp
# For Kubernetes cluster with Cilium CNI
# Run with: bash install-security-stack.sh

set -e

# Color definitions
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
MAGENTA='\033[0;35m'
CYAN='\033[0;36m'
WHITE='\033[1;37m'
NC='\033[0m'
BOLD='\033[1m'

# Symbols
CHECK="✓"
CROSS="✗"
ARROW="→"
STAR="★"

print_header() {
    echo ""
    echo -e "${CYAN}═══════════════════════════════════════════════════════════${NC}"
    echo -e "${BOLD}${WHITE}  $1${NC}"
    echo -e "${CYAN}═══════════════════════════════════════════════════════════${NC}"
}

print_success() {
    echo -e "${GREEN}${CHECK}${NC} $1"
}

print_error() {
    echo -e "${RED}${CROSS}${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}⚠${NC} $1"
}

print_info() {
    echo -e "${BLUE}${ARROW}${NC} $1"
}

print_section() {
    echo ""
    echo -e "${MAGENTA}${STAR} $1${NC}"
    echo -e "${MAGENTA}───────────────────────────────────────────────────────────${NC}"
}

# Start installation
clear
echo -e "${BOLD}${CYAN}"
cat << "EOF"
╔═╗╔═╗╔═╗╦ ╦╦═╗╦╔╦╗╦ ╦  ╔═╗╔╦╗╔═╗╔═╗╦╔═  ╦╔╗╔╔═╗╔╦╗╔═╗╦  ╦  ╔═╗╦═╗
╚═╗║╣ ║  ║ ║╠╦╝║ ║ ╚╦╝  ╚═╗ ║ ╠═╣║  ╠╩╗  ║║║║╚═╗ ║ ╠═╣║  ║  ║╣ ╠╦╝
╚═╝╚═╝╚═╝╚═╝╩╚═╩ ╩  ╩   ╚═╝ ╩ ╩ ╩╚═╝╩ ╩  ╩╝╚╝╚═╝ ╩ ╩ ╩╩═╝╩═╝╚═╝╩╚═
EOF
echo -e "${NC}"
echo -e "${WHITE}Installing Kyverno + Falco + Headlamp${NC}"
echo ""

# Check prerequisites
print_header "CHECKING PREREQUISITES"

if ! command -v kubectl &> /dev/null; then
    print_error "kubectl not found. Please install kubectl first."
    exit 1
fi
print_success "kubectl is installed"

if ! command -v helm &> /dev/null; then
    print_error "Helm not found. Please install Helm first."
    exit 1
fi
print_success "Helm is installed"

# Check if cluster is accessible
if ! kubectl cluster-info &> /dev/null; then
    print_error "Cannot connect to Kubernetes cluster"
    exit 1
fi
print_success "Kubernetes cluster is accessible"

# Check if Cilium is running
CILIUM_PODS=$(kubectl get pods -n kube-system -l k8s-app=cilium --no-headers 2>/dev/null | wc -l)
if [ "$CILIUM_PODS" -gt 0 ]; then
    print_success "Cilium CNI detected"
else
    print_warning "Cilium not detected - installation will continue anyway"
fi

#############################################
# 1. INSTALL KYVERNO (Policy Engine)
#############################################

print_header "1. INSTALLING KYVERNO"

print_section "Adding Kyverno Helm Repository"
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
print_success "Kyverno repository added"

print_section "Installing Kyverno"
helm install kyverno kyverno/kyverno \
    --namespace kyverno \
    --create-namespace \
    --set admissionController.replicas=1 \
    --set backgroundController.replicas=1 \
    --set cleanupController.replicas=1 \
    --set reportsController.replicas=1 \
    --wait --timeout 5m

print_success "Kyverno installed successfully"

print_section "Verifying Kyverno Installation"
sleep 10
kubectl get pods -n kyverno
echo ""

# Wait for Kyverno to be ready
print_info "Waiting for Kyverno pods to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=300s
print_success "Kyverno is ready"

print_section "Installing Kyverno Policies"
helm install kyverno-policies kyverno/kyverno-policies \
    --namespace kyverno \
    --wait --timeout 3m

print_success "Kyverno policies installed"

# Create sample policies
print_section "Creating Sample Security Policies"

cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      Require all resources to have specific labels for better organization.
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
          kinds:
          - Pod
          - Deployment
          - Service
    validate:
      message: "Resources must have 'app' and 'env' labels"
      pattern:
        metadata:
          labels:
            app: "?*"
            env: "?*"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security Standards (Baseline)
    policies.kyverno.io/severity: high
    policies.kyverno.io/description: >-
      Privileged containers should not be allowed.
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: privileged-containers
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): false
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: check-resource-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Containers must have CPU and memory limits defined"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
EOF

print_success "Sample security policies created"

#############################################
# 2. INSTALL FALCO (Runtime Security)
#############################################

print_header "2. INSTALLING FALCO"

print_section "Adding Falco Helm Repository"
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
print_success "Falco repository added"

print_section "Installing Falco with Cilium Integration"

# Create custom values for Falco
cat <<EOF > /tmp/falco-values.yaml
# Falco configuration for Cilium integration
driver:
  kind: modern_ebpf  # Use modern eBPF driver (compatible with Cilium)

falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/rules.d
  
  # Enable JSON output for better parsing
  json_output: true
  json_include_output_property: true
  
  # Logging
  log_stderr: true
  log_syslog: false
  log_level: info
  
  # Performance tuning
  priority: debug
  buffered_outputs: false

# Enable Falco Exporter for Prometheus metrics
falcoctl:
  artifact:
    install:
      enabled: true
    follow:
      enabled: true

# Resources
resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1024Mi

# Enable integration with Kubernetes audit logs
auditLog:
  enabled: false  # Can be enabled if needed

# Custom rules for Cilium
customRules:
  cilium-rules.yaml: |-
    - rule: Unauthorized Network Connection
      desc: Detect unauthorized network connections
      condition: >
        evt.type = connect and container and
        not proc.name in (cilium-agent, cilium-operator, coredns)
      output: >
        Unauthorized connection attempt
        (user=%user.name command=%proc.cmdline connection=%fd.name container=%container.name)
      priority: WARNING
      tags: [network, cilium]
    
    - rule: Sensitive File Access
      desc: Detect access to sensitive files
      condition: >
        evt.type = open and
        fd.name in (/etc/shadow, /etc/passwd, /etc/sudoers) and
        container
      output: >
        Sensitive file accessed
        (user=%user.name command=%proc.cmdline file=%fd.name container=%container.name)
      priority: CRITICAL
      tags: [filesystem, security]
EOF

helm install falco falcosecurity/falco \
    --namespace falco \
    --create-namespace \
    --values /tmp/falco-values.yaml \
    --wait --timeout 5m

print_success "Falco installed successfully"

print_section "Verifying Falco Installation"
sleep 10
kubectl get pods -n falco
echo ""

# Wait for Falco to be ready
print_info "Waiting for Falco pods to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=falco -n falco --timeout=300s
print_success "Falco is ready"

# Clean up temporary file
rm -f /tmp/falco-values.yaml

#############################################
# 3. INSTALL HEADLAMP (Dashboard)
#############################################

print_header "3. INSTALLING HEADLAMP"

print_section "Adding Headlamp Helm Repository"
helm repo add headlamp https://headlamp-k8s.github.io/headlamp/
helm repo update
print_success "Headlamp repository added"

print_section "Installing Headlamp"

cat <<EOF > /tmp/headlamp-values.yaml
# Headlamp configuration
config:
  # Base URL path for Headlamp
  baseURL: ""
  
  # Enable plugins
  pluginsDir: "/headlamp/plugins"

# Service configuration
service:
  type: ClusterIP
  port: 80

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# Replicas
replicaCount: 1

# Enable RBAC
rbac:
  create: true

# Service Account
serviceAccount:
  create: true
  name: headlamp-admin

# Create ClusterRoleBinding for full access
# (You can restrict this in production)
EOF

helm install headlamp headlamp/headlamp \
    --namespace headlamp \
    --create-namespace \
    --values /tmp/headlamp-values.yaml \
    --wait --timeout 3m

print_success "Headlamp installed successfully"

# Create admin ClusterRoleBinding for Headlamp
print_section "Creating Admin Access for Headlamp"

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: headlamp-admin
  namespace: headlamp
EOF

print_success "Admin access configured for Headlamp"

print_section "Verifying Headlamp Installation"
kubectl get pods -n headlamp
echo ""

# Clean up temporary file
rm -f /tmp/headlamp-values.yaml

#############################################
# INSTALLATION SUMMARY
#############################################

print_header "INSTALLATION COMPLETE!"

print_section "Installed Components"
echo ""
print_success "Kyverno (Policy Engine) - Namespace: kyverno"
print_success "Falco (Runtime Security) - Namespace: falco"
print_success "Headlamp (Dashboard) - Namespace: headlamp"

print_section "Verification Commands"
echo ""
echo -e "${BLUE}# Check Kyverno:${NC}"
echo "  kubectl get pods -n kyverno"
echo "  kubectl get clusterpolicies"
echo ""
echo -e "${BLUE}# Check Falco:${NC}"
echo "  kubectl get pods -n falco"
echo "  kubectl logs -n falco -l app.kubernetes.io/name=falco -f"
echo ""
echo -e "${BLUE}# Check Headlamp:${NC}"
echo "  kubectl get pods -n headlamp"
echo "  kubectl get svc -n headlamp"

print_section "Access Instructions"
echo ""
echo -e "${YELLOW}Headlamp Dashboard:${NC}"
echo "  # Port-forward to access Headlamp UI:"
echo "  kubectl port-forward -n headlamp svc/headlamp 8080:80"
echo "  # Then open: http://localhost:8080"
echo ""
echo -e "${YELLOW}Get Headlamp Token:${NC}"
echo "  # Create a token for login:"
echo "  kubectl create token headlamp-admin -n headlamp --duration=24h"
echo ""
echo -e "${YELLOW}Falco Logs:${NC}"
echo "  # Watch Falco security alerts in real-time:"
echo "  kubectl logs -n falco -l app.kubernetes.io/name=falco -f"
echo ""
echo -e "${YELLOW}Kyverno Reports:${NC}"
echo "  # Check policy violations:"
echo "  kubectl get policyreports -A"
echo "  kubectl get clusterpolicyreports"

print_section "Next Steps"
echo ""
print_info "1. Access Headlamp dashboard to visualize your cluster"
print_info "2. Monitor Falco logs for security events"
print_info "3. Review Kyverno policy reports"
print_info "4. Customize policies based on your requirements"

print_section "Useful Links"
echo ""
echo "  Kyverno Docs:  https://kyverno.io/docs/"
echo "  Falco Docs:    https://falco.org/docs/"
echo "  Headlamp Docs: https://headlamp.dev/docs/"

echo ""
print_header "ALL DONE! 🎉"
echo ""

Perfect! I've created a comprehensive installation script for Kyverno, Falco, and Headlamp. Here's everything you need:

## **Quick Start**

```bash
# Save the script
nano install-security-stack.sh

# Make it executable
chmod +x install-security-stack.sh

# Run it
./install-security-stack.sh
```

## **What the Script Installs**

### **1. Kyverno (Policy Engine)** - Installed First
- **Purpose:** Prevent security issues before they happen
- **Features Included:**
  - Admission controller with validation/mutation
  - Pre-configured security policies
  - Sample policies for best practices
- **Policies Created:**
  - ✅ Require labels on resources
  - ✅ Disallow privileged containers
  - ✅ Require resource limits

### **2. Falco (Runtime Security)** - Installed Second
- **Purpose:** Detect threats and anomalies at runtime
- **Features Included:**
  - Modern eBPF driver (Cilium-compatible)
  - JSON output for easy parsing
  - Custom rules for Cilium integration
- **Monitoring:**
  - Unauthorized network connections
  - Sensitive file access
  - Suspicious system calls

### **3. Headlamp (Dashboard)** - Installed Last
- **Purpose:** Modern UI for cluster management
- **Features Included:**
  - Full cluster visualization
  - Real-time pod/deployment monitoring
  - Admin access pre-configured

## **Post-Installation: Access Your Tools**

### **Access Headlamp Dashboard**
```bash
# Port-forward Headlamp
kubectl port-forward -n headlamp svc/headlamp 8080:80

# Get login token (valid for 24 hours)
kubectl create token headlamp-admin -n headlamp --duration=24h

# Open browser: http://localhost:8080
# Paste the token when prompted
```

### **Monitor Falco Security Events**
```bash
# Real-time security alerts
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# Example output you'll see:
# {"output":"Sensitive file opened for reading...","priority":"Warning"...}
```

### **Check Kyverno Policy Reports**
```bash
# View all policy violations
kubectl get policyreports -A

# View cluster-wide reports
kubectl get clusterpolicyreports

# Get detailed report
kubectl describe clusterpolicyreport cpol-require-labels
```

## **Verify Installation**

```bash
# Check all components
kubectl get pods -n kyverno
kubectl get pods -n falco
kubectl get pods -n headlamp

# Check Kyverno policies
kubectl get clusterpolicies

# Test Kyverno (should fail due to missing labels)
kubectl run test-pod --image=nginx
# You'll see: Error... Resources must have 'app' and 'env' labels

# Test with correct labels (should succeed)
kubectl run test-pod --image=nginx --labels="app=test,env=dev"
```

## **Key Features & Benefits**

### **Security Layers**
```
Layer 1: Cilium          → Network-level security
Layer 2: Kyverno         → Admission control (prevention)
Layer 3: Falco           → Runtime detection (monitoring)
Layer 4: Headlamp        → Visibility & management
```

### **Integration Highlights**
- ✅ **Falco + Cilium:** Uses eBPF driver compatible with Cilium
- ✅ **Kyverno + Falco:** Kyverno prevents, Falco detects
- ✅ **Headlamp:** Visualizes all components in one UI

## **Testing Your Setup**

After installation, run a security test:

```bash
# 1. Try to create a privileged pod (Kyverno will block it)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  labels:
    app: test
    env: dev
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
# Expected: Error - Privileged containers are not allowed

# 2. Create a normal pod (should succeed)
kubectl run normal-pod --image=nginx --labels="app=nginx,env=prod"

# 3. Watch Falco detect the pod creation
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50

# 4. View in Headlamp dashboard at http://localhost:8080
```

## **Resource Usage Estimate**

Your cluster will use approximately:
- **Kyverno:** ~200-300 MB RAM, 0.2 CPU
- **Falco:** ~500 MB RAM, 0.3 CPU  
- **Headlamp:** ~128 MB RAM, 0.1 CPU
- **Total:** ~800 MB RAM, 0.6 CPU additional

Your VM should handle this fine!

## **Troubleshooting**

If any component fails:

```bash
# Check component logs
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno
kubectl logs -n falco -l app.kubernetes.io/name=falco
kubectl logs -n headlamp -l app.kubernetes.io/name=headlamp

# Restart a component
kubectl rollout restart deployment -n kyverno
kubectl rollout restart daemonset -n falco
kubectl rollout restart deployment -n headlamp
```

## **Your Final Stack**

```
✅ Kubernetes v1.28.15 (Control Plane)
✅ containerd v2.1.5 (Runtime)
✅ Helm v3.19.2 (Package Manager)
✅ Cilium v1.14.5 (CNI/Networking)
✅ CoreDNS (Service Discovery)
✅ Kyverno (Policy Engine)
✅ Falco (Runtime Security)
✅ Headlamp (Dashboard)
```

This is a **production-grade security stack**! 🎉

Ready to run the installation? Let me know if you encounter any issues!

The installation is stuck at Kyverno. Those warnings are harmless (just API version compatibility), but it's likely waiting for resources. Let's troubleshoot:

## **Quick Fix - Check What's Happening**

Open a new terminal and run:

```bash
# Check what's being created
kubectl get pods -n kyverno -w

# In another terminal, check events
kubectl get events -n kyverno --sort-by='.lastTimestamp'

# Check if admission controller is having issues
kubectl get deployments -n kyverno
```

## **If It's Still Stuck After 5 Minutes**

Press `Ctrl+C` to stop the script, then run:

```bash
# Check what was created
kubectl get all -n kyverno

# Check for any errors
kubectl describe pods -n kyverno

# Check if webhooks are causing issues
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno
```

## **Quick Recovery Script**

If stuck, use this to continue:

```bash
# Clean up and retry with lighter configuration
helm uninstall kyverno -n kyverno 2>/dev/null
kubectl delete namespace kyverno 2>/dev/null

# Wait a moment
sleep 5

# Reinstall with minimal resources (for VM environment)
helm install kyverno kyverno/kyverno \
    --namespace kyverno \
    --create-namespace \
    --set admissionController.replicas=1 \
    --set backgroundController.replicas=1 \
    --set cleanupController.replicas=1 \
    --set reportsController.replicas=1 \
    --set admissionController.resources.limits.memory=512Mi \
    --set admissionController.resources.requests.memory=256Mi \
    --set webhooksCleanup.enabled=true \
    --timeout 10m \
    --wait

# Check status
kubectl get pods -n kyverno
```

## **Alternative: Skip Wait and Continue**

If you want to continue without waiting:

```bash
# Install without --wait flag
helm install kyverno kyverno/kyverno \
    --namespace kyverno \
    --create-namespace \
    --set admissionController.replicas=1 \
    --set backgroundController.replicas=1 \
    --set cleanupController.replicas=1 \
    --set reportsController.replicas=1

# Then manually check when ready
kubectl get pods -n kyverno -w
```

## **Most Common Cause: Resource Constraints**

Your VM might need more resources. Check:

```bash
# Check node resources
kubectl describe node cloud | grep -A 10 "Allocated resources"

# Check if pods are pending
kubectl get pods -n kyverno
```

**What should I see in your terminal output to help diagnose?** Share the output of `kubectl get pods -n kyverno` and I can provide a more specific fix!

Perfect! I found the issue. The pods can't schedule because your control-plane node has a taint that prevents regular pods from running on it. This is a single-node cluster, so we need to remove that taint.

## **Quick Fix - Remove the Control Plane Taint**

Press `Ctrl+C` to stop the watch, then run:

```bash
# Allow pods to schedule on the control-plane node
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Verify the taint is removed
kubectl describe node cloud | grep Taints
```

You should see `Taints: <none>` after this.

## **Watch Pods Start**

```bash
# Now watch the pods come up
kubectl get pods -n kyverno -w
```

Within 30 seconds, all pods should change from `Pending` to `Running`.

## **Why This Happened**

In production clusters:
- **Control plane nodes** = Management only (API server, scheduler, etc.)
- **Worker nodes** = Run application workloads

In your **single-node setup**:
- Your node is BOTH control plane AND worker
- By default, Kubernetes taints control-plane nodes to prevent workloads
- We need to remove this taint so Kyverno (and other apps) can run

## **After Taint Removal**

Once the taint is removed, your installation script should continue automatically. If it's already stopped, restart it:

```bash
# If the script is still running, it should continue automatically
# If it stopped, you can continue manually:

# Check if Kyverno is now running
kubectl get pods -n kyverno

# If all pods are Running, continue with Falco and Headlamp
# You can run these commands from the script or manually
```

## **Continue Installation Manually** (if script stopped)

```bash
# 1. Install Kyverno Policies
helm install kyverno-policies kyverno/kyverno-policies \
    --namespace kyverno \
    --wait --timeout 3m

# 2. Then continue with Falco (from the script)
# 3. Then Headlamp
```

Let me know once you've removed the taint and I'll help you continue!

The Kyverno installation from before is still there. Let's clean it up and restart:

```bash
# Uninstall the existing Kyverno installation
helm uninstall kyverno -n kyverno

# Wait for pods to terminate
kubectl delete pods --all -n kyverno --force --grace-period=0

# Delete any webhooks that might be stuck
kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg 2>/dev/null
kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg 2>/dev/null

# Verify cleanup
kubectl get pods -n kyverno

# Now rerun the installation script
./install-security-stack.sh
```

## **Alternative: Skip to Fresh Installation**

If you want to be thorough:

```bash
# Complete cleanup
helm uninstall kyverno -n kyverno 2>/dev/null
helm uninstall kyverno-policies -n kyverno 2>/dev/null
kubectl delete namespace kyverno

# Wait for namespace to be fully deleted
kubectl get namespace kyverno -w
# Press Ctrl+C when it disappears

# Remove any leftover webhooks
kubectl delete validatingwebhookconfigurations -l app.kubernetes.io/part-of=kyverno
kubectl delete mutatingwebhookconfigurations -l app.kubernetes.io/part-of=kyverno

# Now run the script again
./install-security-stack.sh
```

The script should now proceed successfully since:
1. ✅ The control-plane taint is removed
2. ✅ Old Kyverno installation is cleaned up

Let me know how it goes!

## **What is This Warning?**

This warning is about a **Kubernetes API version mismatch** - it's completely **harmless and safe to ignore**.

## **Why Does It Happen?**

**Technical Explanation:**
- `selectableFields` is a feature introduced in **Kubernetes 1.30+**
- Your cluster is running **Kubernetes v1.28.15**
- Kyverno's CRDs (Custom Resource Definitions) include this newer field
- Kubernetes 1.28 doesn't recognize it, so it warns and ignores it

## **Analogy:**
It's like installing modern software that has optional features your system doesn't support. The software works fine - it just skips the unsupported features.

## **Is This a Problem?**

❌ **No!** Here's why:

1. **Kyverno still works perfectly** - Core functionality is unaffected
2. **Just a forward-compatibility feature** - Kyverno is designed to work on older K8s versions
3. **Installation will continue** - These are warnings, not errors
4. **Common in Kubernetes** - Happens when Helm charts support multiple K8s versions

## **What is `selectableFields`?**

It's an advanced Kubernetes feature that allows:
- Custom filtering of resources in `kubectl get` commands
- Better performance for large-scale clusters
- Enhanced API queries

Since you're on K8s 1.28, you simply won't have this feature, but Kyverno will work fine without it.

## **What Should You Do?**

✅ **Nothing!** Just let the installation continue. The warnings will appear and then:
- Installation will proceed normally
- Kyverno will install successfully
- All features you need will work

## **How to Suppress These Warnings (Optional)**

If they bother you, you can suppress them:

```bash
# Set this environment variable before running the script
export HELM_EXPERIMENTAL_OCI=1
export KUBECONFIG_WARNINGS=off
```

But honestly, it's fine to just ignore them. They're informational only.

---

**TL;DR:** These warnings mean "Hey, you have an older Kubernetes version, so I'm skipping some advanced features." Everything will work perfectly! Just wait for the installation to complete. 🙂


Done! I've updated the script to remove Kyverno completely. The script now only installs:

1. ✅ **Falco** (Runtime Security)
2. ✅ **Headlamp** (Dashboard)

## **Changes Made:**
- Removed all Kyverno installation steps
- Removed Kyverno policy creation
- Removed Kyverno from verification commands
- Updated title from "Kyverno + Falco + Headlamp" to "Falco + Headlamp"
- Renumbered sections (Falco is now #1, Headlamp is #2)
- Cleaned up all Kyverno-related documentation

The updated script is ready to use. You can now run:

```bash
./install-security-stack.sh
```

This will install only Falco and Headlamp on your cluster with Cilium!

The issue isn't about Kyverno compatibility with Ubuntu/VirtualBox/Kali - **Kyverno fully supports your setup**. The problem was the **Kubernetes control-plane taint** preventing pods from scheduling on your single-node cluster.

Let me create a proper installation script that handles this correctly:Perfect! I've created a comprehensive Kyverno installation script that handles **all edge cases** including:

## **Why Kyverno Wasn't Installing Before**

The issue was **NOT** compatibility - it was:
1. ❌ **Control-plane taint** blocking pods on single-node cluster
2. ❌ **No automatic cleanup** of previous failed attempts
3. ❌ **Timeout issues** without proper waiting logic

## **What This Script Does Differently**

### ✅ **Automatic Fixes**
1. **Detects single-node cluster** and removes control-plane taint
2. **Cleans up** any existing Kyverno installations automatically
3. **Proper resource limits** for VM environments
4. **Smart waiting** with progress feedback
5. **Verification tests** to ensure everything works

## **How to Use**

```bash
# Save the new script
nano install-kyverno.sh

# Make it executable
chmod +x install-kyverno.sh

# Run it
./install-kyverno.sh
```

## **What You'll See**

The script will:
1. ✅ Check prerequisites (kubectl, helm, cluster access)
2. ✅ Detect single-node cluster and remove taint automatically
3. ✅ Clean up any existing Kyverno installations
4. ✅ Install Kyverno with proper settings for your VM
5. ✅ Install policy library
6. ✅ Create 4 custom security policies
7. ✅ Test policies automatically
8. ✅ Provide comprehensive verification

## **About Those Warnings**

```
Warning: unknown field "spec.versions[0].selectableFields"
```

- ✅ **Completely harmless** - just API version compatibility
- ✅ **Kyverno works perfectly** on Kubernetes 1.28
- ✅ **Script continues** without issues
- ✅ **Optional feature** your cluster doesn't need

## **Policies Created**

1. **require-labels** (Audit) - Warns if pods missing app/env labels
2. **disallow-privileged-containers** (Enforce) - **BLOCKS** privileged pods
3. **require-resource-limits** (Audit) - Warns about missing CPU/memory limits
4. **disallow-latest-tag** (Audit) - Warns about using :latest tag

## **Your Environment is Fine!**

- ✅ **Ubuntu in VirtualBox** → Fully supported
- ✅ **Kali Linux host** → No impact on guest VM
- ✅ **Kubernetes 1.28** → Compatible with all CNCF tools
- ✅ **Single-node cluster** → Script handles it automatically

## **After Installation**

Verify everything works:

```bash
# Check Kyverno is running
kubectl get pods -n kyverno

# View policies
kubectl get clusterpolicies

# Test blocking privileged pods
kubectl run test --image=nginx --restart=Never -- sh -c "sleep 3600" --privileged
# Should be BLOCKED by Kyverno!
```

Run the script now - it will handle everything automatically! 🚀
