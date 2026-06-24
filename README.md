# Istio 1.22+, Kiali, Prometheus, Grafana, ELK- Setup
Here is the final, complete, and corrected guide, including the missing Helm installation step and refined metric configurations based on the latest official Istio documentation.

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
kubectl create ns istio-system monitoring logging
```

### Phase 3: Install Istio Control Plane
```bash
# Install Istio (default profile includes ingress gateway)
istioctl install --set profile=default -y

# Enable sidecar injection for app workloads
kubectl label namespace default istio-injection=enabled

# Verify ingress gateway is running
kubectl wait --for=condition=ready pod -n istio-system -l istio=ingressgateway --timeout=90s
```

### Phase 4: Install Kiali
```bash
helm repo add kiali https://kiali.org/helm-charts
helm repo update

helm install kiali-server kiali/kiali-server -n istio-system \
  --set auth.strategy=login \
  --set login.username=admin \
  --set login.passphrase=admin \
  --set external_services.prometheus.url=http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090 \
  --set external_services.grafana.url=http://monitoring-grafana.monitoring.svc:80
```

### Phase 5: Install Prometheus & Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set grafana.service.type=ClusterIP
```

### Phase 6: Install ELK Stack
```bash
helm repo add elastic https://helm.elastic.co
helm repo update

# Elasticsearch (Security disabled for simple setup)
helm install elasticsearch elastic/elasticsearch -n logging \
  --set replicas=1 --set xpack.security.enabled=false

# Kibana
helm install kibana elastic/kibana -n logging

# Filebeat (Log shipper)
helm install filebeat elastic/filebeat -n logging
```

### Phase 7: Connect Istio Metrics to External Prometheus
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

### Phase 8: Deploy Test App & Generate Traffic
```bash
# Deploy Bookinfo
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl wait --for=condition=ready pod -l app=productpage --timeout=90s

# Configure Gateway
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Get Ingress Gateway address
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

# Generate traffic
for i in $(seq 1 50); do curl -s -o /dev/null "http://$INGRESS_HOST:$INGRESS_PORT/productpage"; done
```

---

### Phase 9: Access URLs, Paths, and Credentials
*Run these in separate terminal windows to keep the tunnels open.*

#### 1. Kiali (Service Mesh UI)
```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001
```
- **URL:** `http://localhost:20001`
- **User:** `admin` | **Password:** `admin`

#### 2. Grafana (Metrics Dashboards)
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```
- **URL:** `http://localhost:3000`
- **User:** `admin` | **Password:** `prom-operator`

#### 3. Prometheus (Raw Metrics)
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```
- **URL:** `http://localhost:9090`
- **Credentials:** None (Open)

#### 4. Kibana (Logging UI)
```bash
kubectl port-forward -n logging svc/kibana-kibana 5601:5601
```
- **URL:** `http://localhost:5601`
- **Credentials:** None (Security disabled)

---

### Phase 10: Post-Install UI Configurations

#### Configure Kibana (Logs)
1. Go to `http://localhost:5601` → **Management → Stack Management → Kibana → Data Views**.
2. Click **Create data view**.
3. **Name:** `filebeat-*` | **Timestamp field:** `@timestamp`.
4. Save → Go to **Analytics → Discover** to view logs.

#### Configure Grafana (Istio Dashboards)
1. Go to `http://localhost:3000` → **Dashboards → New → Import**.
2. Import these Istio Dashboard IDs (select **Prometheus** data source):
   - **Istio Mesh:** `7639`
   - **Istio Performance:** `11829`
   - **Istio Service:** `7636`
   - **Istio Workload:** `7630`

#### Verify Integration
1. **Prometheus** (`http://localhost:9090`) → **Status → Targets** → Ensure `istiod` and `istio-proxies` are **UP**.
2. **Kiali** (`http://localhost:20001`) → **Graph** → Select `default` namespace → See mesh topology.
3. **Grafana** → Check imported Istio dashboards show traffic data.
