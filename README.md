# Istio 1.22+, Kiali, Prometheus, Grafana- Setup
Here is the final, complete, and corrected guide, including the missing Helm installation step, refined metric configurations, and **persistent storage** setup based on the latest official Istio documentation.

### Phase 1: Install CLI Tools
```bash
# 1. Install Helm
export VERIFY_CHECKSUM=false
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# 2. Install istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
# (Add 'export PATH=$PWD/bin:$PATH' to your ~/.bashrc or ~/.zshrc to persist)
```

### Phase 2: Create Namespaces
```bash
kubectl create ns istio-system
kubectl create ns monitoring
```

### Phase 3: Install Istio Control Plane
```bash
# Install Istio (default profile includes ingress gateway)
istioctl install --set profile=default -y

# Enable sidecar injection for app workloads
kubectl label namespace default istio-injection=enabled
```
```bash
# Verify ingress gateway is running
kubectl get pod -n istio-system
```

### Phase 4: Install Prometheus & Grafana (with Persistent Storage)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set grafana.service.type=ClusterIP
```

If the Grafana pod restarts during/after a reboot, manually imported UI dashboards and metrics data are lost. 
**Fix:** Provision them via Helm with persistent volumes so they automatically recreate and retain data on pod restarts.
#### Check if you have Storage Class setup
```bash
kubectl get sc
```

Create `values.yaml` using `openebs-hostpath` or any other PV provider so we don't accidentally wipe your raw NVMe drive for small monitoring data:

```bash
cat <<EOF > values.yaml
grafana:
  persistence:
    enabled: true
    storageClassName: openebs-hostpath
    size: 1Gi

prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources: { requests: { storage: 2Gi } }

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources: { requests: { storage: 1Gi } }
EOF
```
Label the monitoring namespace to allow privileged pods and hostPath volumes
```
kubectl label --overwrite ns monitoring \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged
```
Then upgrade your release to apply the persistence and dashboards:
```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
```
```bash
# Verify running and PVCs are successfully bound
kubectl get pod -n monitoring
kubectl get pvc -n monitoring
```

### Phase 5: Install Kiali
```bash
helm repo add kiali https://kiali.org/helm-charts
helm repo update
```
```bash
helm install kiali-server kiali/kiali-server -n istio-system \
  --set auth.strategy=anonymous \
  --set external_services.prometheus.url=http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090 \
  --set external_services.grafana.in_cluster_url=http://monitoring-grafana.monitoring.svc:80 \
  --set external_services.grafana.url=http://monitoring-grafana.monitoring.svc:80
```
```bash
# Verify Kiali is running
kubectl get pod -n istio-system
```

### Phase 6: Connect Istio Metrics to External Prometheus
*Note: The `PodMonitor` handles Envoy proxy metrics (including the ingress gateway), while the `ServiceMonitor` handles Istiod control plane metrics.*

```bash
cat <<EOF | kubectl apply -n istio-system -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istiod
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      istio: pilot
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - port: http-monitoring
      interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-proxies
  labels:
    release: monitoring
spec:
  selector:
    matchExpressions:
      - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 15s
EOF
```
---

### Phase 7: Expose Services & Access URLs

First, expose the services by patching them. Choose **either** NodePort (for local/bare-metal clusters) **or** LoadBalancer (for cloud environments).

**Patch Commands:**
#### Option A: Expose via NodePort
```bash
kubectl patch svc -n istio-system kiali -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc -n monitoring monitoring-grafana -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc -n monitoring monitoring-kube-prometheus-prometheus -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc -n monitoring monitoring-kube-prometheus-alertmanager -p '{"spec": {"type": "NodePort"}}'
```

#### Option B: Expose via LoadBalancer
```bash
kubectl patch svc -n istio-system kiali -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc -n monitoring monitoring-grafana -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc -n monitoring monitoring-kube-prometheus-prometheus -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc -n monitoring monitoring-kube-prometheus-alertmanager -p '{"spec": {"type": "LoadBalancer"}}'
```

**Retrieve IPs and Ports:**
```bash
kubectl get svc -n istio-system | grep -E 'kiali'
```
```bash
kubectl get svc -n monitoring | grep -E 'grafana|prometheus|alertmanager'
```

#### 1. Kiali (Service Mesh UI)
- **NodePort URL:** `http://<NODE-IP>:<NODE-PORT>` (Default port: 20001)
- **LoadBalancer URL:** `http://<EXTERNAL-IP>:20001`

#### 2. Grafana (Metrics Dashboards)
- **NodePort URL:** `http://<NODE-IP>:<NODE-PORT>` (Default port: 80)
- **LoadBalancer URL:** `http://<EXTERNAL-IP>`
- **User:** `admin`
Get the password:
```bash
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

#### 3. Prometheus (Raw Metrics)
- **NodePort URL:** `http://<NODE-IP>:<NODE-PORT>` (Default port: 9090)
- **LoadBalancer URL:** `http://<EXTERNAL-IP>:9090`
- **Credentials:** None (Open)

#### 4. Alertmanager (Alert Management UI)
- **NodePort URL:** `http://<NODE-IP>:<NODE-PORT>` (Default port: 9093)
- **LoadBalancer URL:** `http://<EXTERNAL-IP>:9093`
- **Credentials:** None (Open)

---
# 🚀 Deploying & Monitoring an App in `myspace-prod`

Here's how to deploy your application in a new namespace and integrate it with Istio, Kiali, and your observability stack.

---

## Step 1: Create Namespace & Enable Istio Injection

```bash
# 1. Create the namespace
kubectl create namespace myspace-prod

# 2. Enable Istio sidecar injection (this injects Envoy proxies into all pods)
kubectl label namespace myspace-prod istio-injection=enabled

# 3. Label for Pod Security (if your app needs privileged access)
kubectl label namespace myspace-prod \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged
```

---

## Step 2: Deploy Your Application

Deploy your app using standard Kubernetes manifests. The Istio sidecar will be automatically injected.

**Example deployment (`myapp-deployment.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myspace-prod
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:latest  # Replace with your app image
        ports:
        - containerPort: 80
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myspace-prod
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 80
```

**Apply it:**
```bash
kubectl apply -f myapp-deployment.yaml
```

**Verify the sidecar is injected:**
```bash
kubectl get pods -n myspace-prod
# You should see 2/2 READY (app container + istio-proxy sidecar)
```

---

## Step 3: Create Istio Gateway & VirtualService (Optional)

If you want external access to your app via the Istio Ingress Gateway:

```bash
cat <<EOF | kubectl apply -n myspace-prod -f -
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.local"  # Replace with your domain
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - "myapp.local"
  gateways:
  - myapp-gateway
  http:
  - route:
    - destination:
        host: myapp
        port:
          number: 80
EOF
```

---

## Step 4: Monitor with Prometheus & Grafana

### Option A: If Your App Exposes `/metrics` (Prometheus Format)

Create a `ServiceMonitor` to tell Prometheus to scrape your app's metrics:

```bash
cat <<EOF | kubectl apply -n myspace-prod -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
EOF
```

### Option B: View Istio Metrics (Automatic)

Istio automatically generates metrics for all traffic flowing through the sidecar. You don't need to do anything special—just ensure traffic is flowing.

**Generate test traffic:**
```bash
kubectl port-forward -n myspace-prod svc/myapp 8080:80 &
PF_PID=$!
sleep 2
for i in {1..20}; do curl -s http://localhost:8080 > /dev/null; done
kill $PF_PID
```

---

## Step 5: View in Kiali

1. Open **Kiali** (`http://<NODE-IP>:<KIALI_PORT>`)
2. In the top-left dropdown, select **Namespace: myspace-prod**
3. Go to **Graph** → You'll see your app's service mesh topology
4. Go to **Workloads** → Click on `myapp` → See detailed metrics (request rate, error rate, duration)
5. Go to **Services** → Click on `myapp` → See inbound/outbound traffic metrics

---

## Step 6: View in Grafana

### Using the dotdc Kubernetes Dashboards (Already Installed)

1. Open **Grafana** → **Dashboards → Browse**
2. Open **"Kubernetes / Views / Namespaces"**
3. At the top, set:
   - **Datasource:** `Prometheus`
   - **Namespace:** `myspace-prod`
4. You'll see CPU, memory, network, and pod metrics for your namespace

### Using Istio Dashboards

1. Open **"Istio Workload Dashboard"**
2. Set variables:
   - **Datasource:** `Prometheus`
   - **Namespace:** `myspace-prod`
   - **Workload:** `myapp`
3. You'll see request rate, success rate, and latency for your app

### Query Your App's Custom Metrics (If You Have `/metrics`)

1. Go to **Explore** in Grafana
2. Select **Prometheus** as datasource
3. Use PromQL to query your app's metrics:
   ```promql
   rate(http_requests_total{namespace="myspace-prod", app="myapp"}[5m])
   ```

---

## Step 7: Verify Everything is Working

```bash
# 1. Check pods have sidecars
kubectl get pods -n myspace-prod -o wide

# 2. Check Prometheus targets
# Go to Prometheus UI → Status → Targets → Search for "myapp" or "istio-proxies"

# 3. Check Kiali sees the workload
# Go to Kiali → Workloads → Select "myspace-prod" namespace

# 4. Generate traffic and check Grafana
kubectl port-forward -n myspace-prod svc/myapp 8080:80
curl http://localhost:8080
# Then check Grafana dashboards
```

---

## Quick Reference: Common Commands

```bash
# View logs (app + sidecar)
kubectl logs -n myspace-prod deployment/myapp -c myapp
kubectl logs -n myspace-prod deployment/myapp -c istio-proxy

# Exec into the sidecar
kubectl exec -it -n myspace-prod deployment/myapp -c istio-proxy -- /bin/sh

# View Istio proxy config
kubectl exec -n myspace-prod deployment/myapp -c istio-proxy -- pilot-agent request GET config_dump

# Disable sidecar injection (if needed)
kubectl label namespace myspace-prod istio-injection-
kubectl rollout restart deployment myapp -n myspace-prod
```

---

## Troubleshooting

**Issue: Kiali doesn't show my namespace**
- Ensure traffic is flowing through the sidecar (check `kubectl get pods -n myspace-prod` shows 2/2 READY)

**Issue: Grafana shows "No data" for my namespace**
- Check Prometheus targets: `http://<PROMETHEUS-URL>/targets`
- Verify the `ServiceMonitor` is created: `kubectl get servicemonitor -n myspace-prod`

**Issue: App not accessible via Istio Gateway**
- Check Gateway/VirtualService: `kubectl get gateway,virtualservice -n myspace-prod`
- Verify the host matches what you're sending in the HTTP request

---
# Uninstallation
Cleanup guide to completely uninstall and remove all traces of the setup. 

Run these in order to avoid dependency errors.

### 1. Uninstall Helm Releases (Monitoring)
```bash
# Uninstall Prometheus/Grafana stack
helm uninstall monitoring -n monitoring

# Uninstall Kiali
helm uninstall kiali-server -n istio-system
```

### 2. Uninstall Istio Control & Data Plane
```bash
# Purge removes all Istio components, gateways, and CRDs
istioctl uninstall --purge -y
```

### 3. Delete Namespaces (Cleans up lingering pods, configs, and PVCs)
```bash
kubectl delete ns istio-system monitoring
```

### 4. Remove Sidecar Injection Labels
```bash
# Remove from default namespace (add others if you labeled them)
kubectl label namespace default istio-injection-
```

### 5. Clean Up Local Files & CLIs (Optional)
```bash
# Remove downloaded Istio directory and istioctl binary
rm -rf ~/istio-*

# Remove Helm binary (if you want to completely remove it)
sudo rm /usr/local/bin/helm
```

**Note on Storage Cleanup:** 
Deleting the namespaces in Step 3 will automatically delete the Persistent Volume Claims (PVCs). However, because you are using OpenEBS local provisioners (`openebs-hostpath` or `local-nvme`), the actual data remains on the node's disk. To fully reclaim the space, you must manually delete the leftover folders on your nodes. For OpenEBS, these are typically located at:
```bash
# Run this on the worker nodes where the pods were scheduled
sudo rm -rf /var/openebs/local/*
```
