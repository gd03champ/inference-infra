# Monitoring (Prometheus, Grafana, GPU and serving metrics)

Observability for three layers: the **cluster** (kube-prometheus-stack), the **GPU**
(DCGM exporter), and **serving** (vLLM pod metrics plus llm-d EPP metrics). It all lands
in Prometheus and shows up in Grafana at `watchme.gd03.me`.

## Files

| File | What it is |
|------|-----------|
| `promstack-values.yaml` | Helm values for kube-prometheus-stack (gp3-backed PVCs) |
| `dcgm-exporter-values.yaml` | Helm values for the NVIDIA DCGM exporter (GPU metrics) |
| `httproute-grafana.yaml` | HTTPRoute exposing Grafana on `watchme.gd03.me` |
| `llm-d-epp-dashboard.json` | Custom Grafana dashboard for llm-d EPP (router) metrics |

The default StorageClass ([`gp3-storageclass.yaml`](gp3-storageclass.yaml)) has to
exist first so the Prometheus and Grafana PVCs can bind.

---

## 1. EBS CSI driver (for PVCs)

kube-prometheus-stack persists Prometheus and Grafana to EBS volumes, so the EBS CSI
driver needs to be installed and permitted via Pod Identity (same pattern as Karpenter).

```bash
aws iam create-role --role-name AmazonEKS_EBS_CSI_DriverRole-${CLUSTER_NAME} \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}'

aws iam attach-role-policy --role-name AmazonEKS_EBS_CSI_DriverRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

aws eks create-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-ebs-csi-driver \
  --pod-identity-associations serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole-${CLUSTER_NAME}

# default StorageClass
kubectl apply -f ../gp3-storageclass.yaml
```

## 2. kube-prometheus-stack

Node exporters are turned off here because the EKS Prometheus Node Exporter addon already
covers them. Prometheus and Grafana use gp3 PVCs.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f promstack-values.yaml
```

One convention to internalize: every ServiceMonitor and PodMonitor in this repo carries
`labels.release: kube-prometheus-stack`. That label is what this Prometheus selects on.
Leave it off and your metrics quietly never get scraped.

## 3. DCGM exporter (GPU metrics)

Runs only on GPU nodes (`nvidia.com/gpu.present: "true"`) and tolerates the
`llm-d.ai/role` taint so it can land on the tainted GPU nodes.

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm upgrade --install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  -n monitoring -f dcgm-exporter-values.yaml
```

This gives you per-GPU utilization, memory, temperature, and power in Grafana.

## 4. Expose Grafana

```bash
kubectl apply -f httproute-grafana.yaml   # watchme.gd03.me -> kube-prometheus-stack-grafana:80
```

## 5. Serving metrics

- **Standalone vLLM** deployments carry their own ServiceMonitor (see
  [`../vllm-qwen3.6-4b.yaml`](../vllm-qwen3.6-4b.yaml)).
- **llm-d model servers** get a PodMonitor from the kustomize `components/monitoring`
  overlay. It needs the `release: kube-prometheus-stack` label added to
  `decode-podmonitor.yaml`, then re-apply the overlay. Details in
  [`../infra-qwen36/`](../infra-qwen36/).
- **llm-d router/EPP** metrics are enabled in the router Helm values
  (`monitoring.prometheus.enabled: true`).

## 6. EPP dashboard

Import `llm-d-epp-dashboard.json` into Grafana (Dashboards then Import) for llm-d EPP
metrics: queue size, routing, per-pool stats. I built this one because there was no good
public dashboard for it, and it is the same signal KEDA scales on, so you can watch the
autoscaling input and decision together.
