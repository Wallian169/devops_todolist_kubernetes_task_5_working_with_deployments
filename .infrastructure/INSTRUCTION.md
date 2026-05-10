## How to deploy the app to k8s

- Apply Namespace first
```bash
kubectl apply -f .infrastructure/namespace.yml
```

-  Apply Service, Deployment, and HPA (in that order)
```bash
kubectl apply -f .infrastructure/nodeport.yml
kubectl apply -f .infrastructure/deployment.yml
kubectl apply -f .infrastructure/hpa.yml
```

- (Optional) Apply BusyBox for debugging
```bash
kubectl apply -f .infrastructure/busybox.yml
```

## Configuration Rationale

### 1. Resource Requests and Limits
- **Memory:** 128Mi (Request) / 256Mi (Limit)
- **CPU:** 250m (Request) / 500m (Limit)

**Reasoning:**
- **Requests:** Define guaranteed resources. 128Mi/250m is sufficient for an idle Django app to handle basic requests.
- **Limits:** Cap maximum consumption to prevent resource starvation on the node during traffic spikes.

### 2. HPA Configuration Choice
- **Minimum Replicas:** 2
- **Maximum Replicas:** 5
- **Scaling Metrics:** CPU (70%) and Memory (80%)

**Why this configuration?**
- **High Availability (Min 2):** By setting the minimum to 2, we ensure that if one pod fails, the application remains available. This eliminates the risk of a single point of failure for a simple web app.
- **Cost vs. Performance (Max 5):** While we could scale to 10+, setting a limit of 5 balances performance with cluster cost. It allows handling moderate traffic surges without consuming excessive cluster resources or risking node instability.
- **Dual Metrics (CPU + Memory):** Scaling based on both ensures the app can handle different load patterns. For example, a database query might spike memory usage before CPU usage. Ignoring memory could lead to Out-Of-Memory (OOM) kills without scaling up.
- **Thresholds (70/80):** These thresholds provide a "buffer" to prevent "thrashing" (rapidly scaling up and down). If usage drops slightly, the system won't immediately scale out again.

### 3. Update Strategy Configuration
- **Type:** RollingUpdate
- **maxUnavailable:** 0
- **maxSurge:** 1

**Why these numbers?**
- **maxUnavailable: 0:** This is critical for maintaining 100% availability. It instructs Kubernetes to wait for a new pod to become fully ready *before* terminating an old pod. This ensures that at least `replicas` (2) pods are always serving traffic during an update.
- **maxSurge: 1:** This allows Kubernetes to spin up one extra pod temporarily. During an update, the process goes: `Create New Pod -> New Pod Ready -> Kill Old Pod`. This ensures we never drop below 2 pods during the update process.

### 4. Accessing the App
After deployment, you can access the application using the **NodePort** service configuration defined in `nodeport.yml`:

1. **Identify the Node IP:**
   Get the external IP of one of the nodes in your cluster:
   ```bash
   kubectl get nodes -o wide
   # Note the INTERNAL-IP of one of the nodes (e.g., 192.168.1.10)
   ```

2. **Access URL:**
   Open a browser and navigate to:
   ```
   http://<NODE-IP>:30080/
   ```
   Replace `<NODE-IP>` with the actual IP address of the cluster node.
   
3. **API Access:**
   To access the API, append `/api/` to the URL:
   ```
   http://<NODE-IP>:30080/api/
   ```

> **Note:** The service is exposed on port **30080** (NodePort). If you plan to use an external Load Balancer or Ingress Controller, you would map a public domain to the internal `todoapp` service (port 80) instead of using the NodePort directly.
