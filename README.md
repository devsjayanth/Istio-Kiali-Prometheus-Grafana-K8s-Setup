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
  dashboards:
    istio:
      mesh: { gnetId: 7639, revision: 1, datasource: Prometheus }
      performance: { gnetId: 11829, revision: 1, datasource: Prometheus }
      service: { gnetId: 7636, revision: 1, datasource: Prometheus }
      workload: { gnetId: 7630, revision: 1, datasource: Prometheus }

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

### Phase 10: Post-Install UI Configurations

#### Configure Grafana (Istio Dashboards)
*Note: Because we provisioned these in the `values.yaml`, they should already be imported. If not, you can manually import them:*
1. Go to the **Grafana URL** → **Dashboards → New → Import**.
2. Import these Istio Dashboard IDs (select **Prometheus** data source):
   - **Istio Mesh:** `7639`
   - **Istio Performance:** `11829`
   - **Istio Service:** `7636`
   - **Istio Workload:** `7630`

#### Verify Integration
1. **Prometheus** → **Status → Targets** → Ensure `istiod` and `istio-proxies` are **UP**.
2. **Kiali** → **Graph** → Select `default` namespace → See mesh topology.
3. **Grafana** → Check imported Istio dashboards show traffic data.
4. **Alertmanager** → **Alerts** tab → Verify active alerts and silences are displaying correctly.

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
