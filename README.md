# Kubernetes Logging Stack with Loki & Grafana

A complete demo/learning setup for log aggregation and visualization using Loki, Grafana Alloy, and Grafana on Kubernetes.

##  Overview

This project demonstrates a production-like logging pipeline with:
- **Fake log generation** using `flog`
- **Log collection** with Grafana Alloy
- **Log storage** in Loki
- **Visualization** in Grafana with pre-configured dashboards

Perfect for learning, testing, or demonstrating logging infrastructure concepts.

##  Architecture

```
┌─────────────────┐
│  Log Generator  │ (flog)
│  Fake JSON Logs │
└────────┬────────┘
         │ writes to
         │ /var/log/generated-logs.txt
         ▼
┌─────────────────┐
│ Shared PVC      │
│ (ReadWriteMany) │
└────────┬────────┘
         │ mounted by
         ▼
┌─────────────────┐
│  Grafana Alloy  │
│  - Tail logs    │
│  - Parse JSON   │
│  - Extract      │
│    labels       │
└────────┬────────┘
         │ pushes to
         ▼
┌─────────────────┐
│  Loki           │
│  (SingleBinary) │
│  - Stores logs  │
│  - Indexes      │
│    labels       │
└────────┬────────┘
         │ queried by
         ▼
┌─────────────────┐
│  Grafana        │
│  - Dashboards   │
│  - Explore      │
│  - LogQL        │
└─────────────────┘
```

## ✨ Features

- ✅ Automatic JSON log parsing
- ✅ Label extraction (`http_method`, `http_status`)
- ✅ Pre-configured Grafana dashboards
- ✅ Real-time log streaming
- ✅ Ready-to-use LogQL queries
- ✅ Persistent storage for logs and dashboards
- ✅ Single-namespace deployment

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (minikube, kind, or any K8s cluster)
- `kubectl` configured
- `helm` 3.x installed
- **Storage class that supports ReadWriteMany** (important!)

### Installation

1. **Clone the repository**
   ```bash
   git clone <your-repo-url>
   cd <repo-name>
   ```

2. **Create the monitoring namespace**
   ```bash
   kubectl create namespace monitoring
   ```

3. **Deploy Loki**
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   helm install loki grafana/loki -n monitoring -f loki-values.yaml
   ```

4. **Wait for Loki to be ready**
   ```bash
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=loki -n monitoring --timeout=300s
   ```

5. **Deploy the logging stack**
   ```bash
   kubectl apply -f fake-logs.yaml -n monitoring
   ```

6. **Deploy Grafana**
   ```bash
   helm install grafana grafana/grafana -n monitoring -f grafana-values.yaml
   ```

7. **Wait for all pods to be ready**
   ```bash
   kubectl get pods -n monitoring -w
   ```

### Access Grafana

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open http://localhost:3000

**Default credentials:**
- Username: `admin`
- Password: `kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`  

## 📊 Using the Dashboard

### Pre-loaded Dashboard

1. Navigate to **Dashboards** → **Browse**
2. Open **Generated Logs Dashboard**
3. View:
   - Live log stream
   - HTTP methods over time
   - HTTP status codes distribution

## If not Dashboard Loaded 

Add data source for Loki in Grafana : select **Loki** as the data source type.
- URL: `http://loki-gateway.monitoring.svc.cluster.local`
- Access: Proxy
- JSON Data: `{"maxLines": 1000}`

### Explore Logs

Go to **Explore** and try these LogQL queries:

**View all logs:**
```logql
{job="generated-logs"}
```

**Filter by HTTP method:**
```logql
{job="generated-logs"} | json | http_method="GET"
```

**Filter by status code:**
```logql
{job="generated-logs"} | json | http_status="500"
```

**Show only errors (5xx):**
```logql
{job="generated-logs"} | json | http_status=~"5.."
```

**Count requests by status:**
```logql
sum by (http_status) (count_over_time({job="generated-logs"} [1m]))
```

**Request rate by method:**
```logql
sum by (http_method) (rate({job="generated-logs"} [1m]))
```

**Top 10 slowest requests:**
```logql
topk(10, sum by (request) (count_over_time({job="generated-logs"} [5m])))
```

##  Configuration

### Loki Configuration (`loki-values.yaml`)

- **Deployment Mode:** SingleBinary (simple, all-in-one)
- **Storage:** Filesystem (no object storage needed)
- **Auth:** Disabled (for demo purposes)
- **Replication:** 1 replica
- **Resources:** Minimal (adjustable)

### Alloy Configuration

Configured to:
- Tail log file: `/var/log/generated-logs.txt`
- Parse JSON logs
- Extract labels: `http_method`, `http_status`
- Forward to Loki gateway

### Log Generator

- **Image:** `mingrammer/flog:0.4.3`
- **Format:** JSON
- **Rate:** 10 logs per second
- **Output:** `/var/log/generated-logs.txt`

## 📁 Repository Structure

```
.
├── README.md                    # This file
├── loki-values.yaml             # Loki Helm values
├── grafana-values.yaml          # Grafana Helm values with datasource
├── fake-logs.yaml               # Log generator + Alloy deployment

```

## 🛠️ Troubleshooting

### Logs not appearing in Grafana

1. **Check if logs are being generated:**
   ```bash
   kubectl exec -n monitoring <alloy pod> -- cat /var/log/generated-logs.txt | head -10
   ```

2. **Check Alloy is tailing the file:**
   ```bash
   kubectl logs -n monitoring -l app=alloy | grep "tail routine"
   ```

3. **Verify Loki is receiving logs:**
   ```bash
   kubectl port-forward -n monitoring svc/loki-gateway 3100:80
   curl -s "http://localhost:3100/loki/api/v1/labels" | jq
   ```

4. **test logs for loki:**
   
Can be found in `test.txt`


### PVC Mounting Issues

If you see `unbound immediate PersistentVolumeClaims` error:

- Your cluster needs a storage class supporting **ReadWriteMany**
- For minikube, you might need to use hostPath provisioner
- Consider using a different storage solution (NFS, Ceph, etc.)

### Alloy not finding log file

**Symptom:** `stat /var/log/generated-logs.txt: no such file or directory`

**Solution:** Restart Alloy after log-generator is running:
```bash
kubectl rollout restart deployment alloy -n monitoring
```

### Grafana datasource connection fails

**Check Loki is accessible:**
```bash
kubectl exec -n monitoring -l app.kubernetes.io/name=grafana -- \
  wget -qO- http://loki-gateway.monitoring.svc.cluster.local/ready
```

## 📈 Scaling & Production Considerations

This setup is designed for **demo/learning purposes**. For production:

- [ ] Enable authentication on Loki
- [ ] Use distributed Loki deployment mode
- [ ] Configure object storage (S3, GCS, etc.)
- [ ] Set up proper RBAC
- [ ] Add retention policies
- [ ] Configure backup strategies
- [ ] Use Ingress for external access
- [ ] Enable TLS/HTTPS
- [ ] Set resource limits appropriately
- [ ] Monitor with Prometheus
- [ ] Set up alerting rules

## 🔄 Updating the Stack

### Update Loki
```bash
helm upgrade loki grafana/loki -n monitoring -f loki-values.yaml
```

### Update Grafana
```bash
helm upgrade grafana grafana/grafana -n monitoring -f grafana-values.yaml
```

### Update Logging Components
```bash
kubectl apply -f fake-logs.yaml -n monitoring
```

## 🗑️ Cleanup

**Remove everything:**
```bash
# Uninstall Helm charts
helm uninstall loki -n monitoring
helm uninstall grafana -n monitoring

# Delete Kubernetes resources
kubectl delete -f fake-logs.yaml -n monitoring

# Delete namespace (optional)
kubectl delete namespace monitoring
```

## 📚 Resources

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 🙋 Support

If you encounter any issues or have questions:
1. Check the [Troubleshooting](#-troubleshooting) section
2. Review the logs: `kubectl logs -n monitoring <pod-name>`
3. Open an issue in this repository

## ⭐ Acknowledgments

- **Grafana Labs** for Loki and Grafana
- **mingrammer** for the flog fake log generator
- Kubernetes community for the amazing ecosystem

---

## Quick Reference

**To deploy everything:**
```bash
# 1. Deploy Loki
helm install loki grafana/loki -n monitoring -f loki-values.yaml

# 2. Deploy log generator & Alloy
kubectl apply -f fake-logs.yaml -n monitoring

# 3. Deploy Grafana
helm install grafana grafana/grafana -n monitoring -f grafana-values.yaml
```

**To access Grafana:**
```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
# Open http://localhost:3000 (admin/admin)
```

**Happy Logging!🎉**