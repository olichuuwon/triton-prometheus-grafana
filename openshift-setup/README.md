
# Guide to OpenShift Setup

## Updated Setup Instructions for Prometheus and Grafana

### **Step 1: Deploy Prometheus**

To deploy Prometheus manually instead of using OpenShift's built-in operator:

1. Apply the Prometheus Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: triton-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
          - "--config.file=/etc/prometheus/prometheus.yml"
          - "--storage.tsdb.path=/prometheus"
        ports:
          - containerPort: 9090
        volumeMounts:
          - name: prometheus-config
            mountPath: /etc/prometheus
          - name: prometheus-data
            mountPath: /prometheus
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
        - name: prometheus-data
          emptyDir: {}

```

2. Create a Prometheus Service to expose it within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: triton-monitoring
spec:
  selector:
    app: prometheus
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
  type: ClusterIP

```

3. Expose Prometheus externally using a Route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: prometheus
  namespace: triton-monitoring
spec:
  to:
    kind: Service
    name: prometheus
    weight: 100
  port:
    targetPort: 9090
  tls:
    termination: edge
  wildcardPolicy: None

```

4. Configure Prometheus with a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: triton-monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 10s # By default, scrape targets every 10 seconds.

    scrape_configs:
      # Scraping configuration for Triton inference service
      - job_name: 'triton'
        scrape_interval: 5s
        static_configs:
          - targets: ['monitoring-triton-inference-services.apps.nebula.sl']

      # Additional example scrape jobs (commented out)
      # - job_name: 'node_exporter'
      #   static_configs:
      #     - targets: ['node_exporter:9100']

      # - job_name: 'cadvisor'
      #   static_configs:
      #     - targets: ['cadvisor:8080']

```

---

### **Step 2: Deploy Grafana**

To visualize the metrics collected by Prometheus:

1. Apply the Grafana Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: triton-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: docker.io/grafana/grafana-oss:11.2.2
          ports:
            - containerPort: 3000
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-data
          env:
            - name: GF_SECURITY_ADMIN_USER
              value: "admin"
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"
      volumes:
        - name: grafana-data
          emptyDir: {}
```

2. Create a Grafana Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: triton-monitoring
spec:
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
```

3. Expose Grafana externally using a Route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: grafana
  namespace: triton-monitoring
spec:
  to:
    kind: Service
    name: grafana
  port:
    targetPort: 3000
```

---

### **Step 3: Configure Prometheus as a Data Source in Grafana**

Once Grafana is deployed and accessible:

1. Access Grafana using the route URL from your cluster.
2. Log in using the credentials defined in the deployment (`admin/admin` by default).
3. Navigate to **Configuration > Data Sources > Add Data Source**.
4. Select **Prometheus** and provide the Prometheus service URL, e.g.:
   ```
   http://prometheus.triton-monitoring.svc:9090
   ```

---

### **Step 4: Verify and Create Dashboards**

- Verify Prometheus is collecting metrics by checking its targets:
  Visit the Prometheus UI (via route) and ensure all targets are listed as "UP."
- Create Grafana dashboards using queries such as:
  - **Total HTTP Requests**:
    ```promql
    http_requests_total
    ```
  - **CPU Usage**:
    ```promql
    rate(container_cpu_usage_seconds_total[1m])
    ```

---

## Optional: ServiceMonitor Setup (For Reference)

The following outlines the original `ServiceMonitor` setup, which can be revisited later.

1. **ServiceMonitor YAML**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: servicemonitor
  namespace: triton-monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: triton-is-predictor
  endpoints:
    - port: metrics
      interval: 5s
      path: /metrics

```

2. **Prometheus Configuration to Include ServiceMonitor**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceMonitorSelector:
    matchLabels:
      release: prometheus
```

3. **Service with Correct Labels**:
Ensure the service exposing `/metrics` includes labels matching the `selector.matchLabels` in the `ServiceMonitor`. Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  labels:
    app: example-app
spec:
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
  selector:
    app: example-app
```

---

## Notes

- The standalone Prometheus and Grafana deployment ensures greater control and stability.
- Retain the ServiceMonitor setup as a backup reference.

---
