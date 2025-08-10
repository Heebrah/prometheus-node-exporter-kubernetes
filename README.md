## Kubernetes Cluster with Prometheus
Prometheus is an open-source monitoring system that collects metrics from applications and infrastructure. In a Kubernetes cluster, Prometheus is commonly deployed to monitor cluster health, workloads, and resource usage.

It works by scraping metrics from various Kubernetes components and exporters such as:

* **kube-apiserver** – cluster control plane metrics
* **kubelet** – node-level metrics
* **node-exporter** – OS-level metrics
* **kube-state-metrics** – Kubernetes object state metrics

When deployed inside the cluster, Prometheus uses Kubernetes service discovery to automatically detect pods, nodes, and services to monitor. Metrics are stored in a time-series database and can be queried using PromQL for analysis or visualized with tools like Grafana.

This setup provides real-time visibility into cluster performance, resource consumption, and system health, enabling proactive alerting and troubleshooting.



# Kubernetes Monitoring on AWS EKS — Step-by-Step Deployment & Maintenance

## Prerequisites

* AWS CLI installed and configured (`aws configure`)

i. download cli
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
![caption](/img/5.install-cli.jpg)

ii. unzip with
```
unzip awscli.zip
```
![caption](/img/6.unzip-aws-cli.jpg)
iii. install
```
sudo ./aws/install
```
![caption](/img/7.aws-install.jpg)
confirm installation
```
iv. aws --version
```
![caption](/img/8.aws-cli-version.jpg)

* `kubectl` installed and configured
```
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

![caption](/img/10.install-kubernetes.jpg)

* Access to an existing EKS cluster or ability to create one
i. to check if you have a cluster use the command below and you can input your own aws region
```
aws eks list-clusters --region --us-east-2
```
![caption](/img/11.check-my-cluster.jpg)
and if we don't we can create one from the console or other means. 
ii. From the console search EKS (Elastic Kubernetes service) and create cluster
![caption](/img/12.configure-cluster.jpg)

* Helm installed (optional, but recommended)
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus -n monitoring
```

---

## 1. Connect to Your EKS Cluster

```bash
aws eks --region <region> update-kubeconfig --name <cluster-name>
kubectl get nodes
```
![caption](/img/16.cluster-now-active.jpg)

Verify the cluster nodes show up.

---

## 2. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```
![caption](/img/19.create-monitoring.jpg)
---

## 3. Deploy Node Exporter DaemonSet
i. Edit `node-exporter-daemonset.yaml` to ensure:

```yaml
securityContext:
  runAsUser: 65534
  runAsNonRoot: true
```

ii. Apply the manifest:

```bash
kubectl apply -f node-exporter-daemonset.yaml -n monitoring
```

![caption](/img/20.run-kubectl-daemon.jpg)

iii. Confirm pods are running:

```bash
kubectl get daemonsets -n monitoring
kubectl get pods -n monitoring -l app=node-exporter
```
![caption](/img/25.get-pod-monitor.jpg)
---



## 4. Edit the config to scrape Kubernetes**

i. Open `prometheus.yml` and replace or add scrape jobs for your Kubernetes services.
For example, if you’ve deployed **node-exporter** in your cluster:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app]
        action: keep
        regex: node-exporter

```
![caption](/img/47.sudo-nano-kubectl.jpg)

ii. Reload the prometheus
using
```
sudo systemctl restart prometheus
```

## 5. Access Metrics Locally

![caption](/img/30.run-kubectl.jpg)
i. Port-forward Node Exporter:

```bash
kubectl port-forward -n monitoring pod/<node-exporter-pod> 19100:9100
```

ii. Open browser at:

```
http://localhost:19100/metrics
```

iii. Port-forward Prometheus:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:9090
```

iv. Access Prometheus UI:

```
http://localhost:9090
```

---

## 6. Query Prometheus Metrics

Examples:

i. CPU usage per node:

```promql
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```
![caption](/img/48.rate-cpu.jpg)

ii. Memory usage per node:

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```
![caption](/img/40.memory-bytes.jpg)
---
iii. file system
![caption](/img/41.node-file.jpg)


iv. total network recieve
![caption](/img/42.node-network.jpg)

Thank You.!!!
