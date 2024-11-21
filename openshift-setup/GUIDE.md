### **Step 1: Verify the ServiceMonitor Setup**
Your `ServiceMonitor` setup ensures Prometheus in OpenShift can scrape metrics from your application. Ensure the following:
1. The `selector.matchLabels` matches the labels on the service exposing `/metrics`.
2. The `endpoints` correctly define the `port` and `path`.

#### **Your `ServiceMonitor` YAML (adjust if needed):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: triton-servicemonitor
  namespace: starchat
  labels:
    release: prometheus  # This should match the Prometheus operator's selector
spec:
  selector:
    matchLabels:
      app: triton-is-predictor
  endpoints:
    - port: metrics
      interval: 5s
      path: /metrics
```

#### **Ensure the Service Exists:**
Your service should have the `app: triton-is-predictor` label and expose the `metrics` port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: triton-service
  namespace: starchat
  labels:
    app: triton-is-predictor
spec:
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
  selector:
    app: triton-is-predictor
```

---

### **Step 2: Configure Prometheus to Scrape Metrics**
Prometheus in OpenShift uses the `ServiceMonitor` resource to discover scrape targets. Verify that Prometheus is configured with a `serviceMonitorSelector` to include your `ServiceMonitor`:

#### **Example Prometheus Configuration:**
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

---

### **Step 3: Deploy Grafana**
In an OpenShift environment, you’ll typically deploy Grafana as a pod using a Deployment or StatefulSet.

#### **Example Grafana Deployment for OpenShift:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
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

#### **Expose Grafana as a Route:**
Create an OpenShift route to expose Grafana:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: grafana
  namespace: monitoring
spec:
  to:
    kind: Service
    name: grafana
  port:
    targetPort: 3000
```

---

### **Step 4: Add Prometheus as a Data Source**
Once Grafana is running:
1. Access Grafana using the OpenShift route you created.
2. Log in with the credentials (`admin/admin` in this example).
3. Add Prometheus as a data source:
   - Navigate to **Configuration > Data Sources > Add data source**.
   - Select **Prometheus**.
   - Enter the Prometheus service URL. For example:
     ```
     http://prometheus-operated.monitoring.svc:9090
     ```

---

### **Step 5: Create Dashboards**
Create a Grafana dashboard to visualize the metrics scraped by your `ServiceMonitor`.

#### **Example PromQL Queries:**
- **Total HTTP Requests**:
  ```promql
  http_requests_total{namespace="starchat", pod=~"triton.*"}
  ```
- **CPU Usage**:
  ```promql
  rate(container_cpu_usage_seconds_total{namespace="starchat", pod=~"triton.*"}[1m])
  ```
- **Memory Usage**:
  ```promql
  container_memory_usage_bytes{namespace="starchat", pod=~"triton.*"}
  ```

---

### **Step 6: Verify the Setup**
- **Check Prometheus Targets**:
  Open Prometheus at `http://prometheus-operated.monitoring.svc:9090/targets` to verify the `ServiceMonitor` is active and targets are "UP."
- **Test Grafana Panels**:
  Add panels to your dashboard with PromQL queries to confirm data is displayed correctly.

---
