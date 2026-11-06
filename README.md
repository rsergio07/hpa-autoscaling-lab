# hpa-autoscaling-lab

Hit a shop with a traffic spike, measure what customers feel while the Horizontal Pod Autoscaler reacts, and test whether tuning really helps.

**Video walkthrough:** on the [Reliability Bench](https://www.youtube.com/@reliabilitybench) YouTube channel

## What you will learn

- How the HPA makes decisions
- Why CPU requests matter
- Why autoscaling is not instant
- How to tune minReplicas and scaling behavior

**SLO:** 99% of checkouts finish in under 500ms.

## Requirements

Docker with 4 CPUs and 8 GiB of memory, plus `git`, `make`, `kind`, `kubectl`, `helm`, `jq` and `curl`. Open source only, and it runs locally. The first run downloads several images.

Tested on one setup: macOS arm64, Colima 0.10.3 (4 CPUs, 8 GiB, `vz`), kind 0.33.0 with Kubernetes 1.37, Helm 4.3.1 and kube-prometheus-stack 92.18.0. On Colima, first run:

```bash
colima ssh -- sudo sysctl -w fs.inotify.max_user_instances=1024 fs.inotify.max_user_watches=1048576
```

## The lab

Clone the repository, then run every command from its root.

```bash
git clone https://github.com/rsergio07/hpa-autoscaling-lab.git hpa-autoscaling-lab
cd hpa-autoscaling-lab
```

### Step 1: Check your machine

Every line should say `ok`.

```bash
make preflight
```

### Step 2: Create the cluster

The private `.kubeconfig` keeps this lab away from your other clusters. Run the `export` again in any new terminal.

```bash
kind create cluster --config cluster/kind.yaml --kubeconfig .kubeconfig
export KUBECONFIG=$PWD/.kubeconfig
kubectl get nodes
```

### Step 3: Build the app and load it into the cluster

kind nodes have their own container runtime, so the image must be loaded into them.

```bash
docker build -t shop:lab app
kind load docker-image shop:lab --name lab
```

### Step 4: Install monitoring

This installs Prometheus and Grafana with only what the lab needs.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack --version 92.18.0 -n monitoring --create-namespace -f deploy/monitoring-values.yaml --wait --timeout 8m
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server --version 3.14.0 -n kube-system --set args={--kubelet-insecure-tls} --wait --timeout 3m
```

### Step 5: Deploy the shop

Grafana is at <http://localhost:30300/d/lab> with no login.

```bash
kubectl apply -f deploy/dashboard.yaml
kubectl apply -f deploy/shop.yaml
kubectl -n shop rollout status deployment/storefront --timeout=2m
kubectl -n shop get pods
```

### Step 6: Add the autoscaler

The target is 60% of the CPU request. Each checkout burns 15ms of CPU, and each pod is limited to 200m.

```bash
cat deploy/hpa.yaml
kubectl apply -f deploy/hpa.yaml
# Wait 90 seconds
kubectl -n shop get hpa
./scripts/verdict.sh 60
```

### Step 7: Send a traffic spike

Traffic goes from 5 to 40 checkouts per second.

```bash
kubectl -n shop set env deployment/customers RPS=40
# Wait 40 seconds
kubectl -n shop get hpa
kubectl -n shop get pods -l app=storefront
# Wait 60 seconds
./scripts/verdict.sh 60
# Wait 80 seconds
kubectl -n shop get hpa
./scripts/verdict.sh 30
```

### Step 8: Return to normal traffic

Scale down is slow by design.

```bash
kubectl -n shop set env deployment/customers RPS=5
# Wait 30 seconds
kubectl -n shop get hpa
```

### Step 9: Tune the autoscaler

Keep spare capacity with `minReplicas: 3`, lower the target, and allow faster scale up.

```bash
cat deploy/hpa-tuned.yaml
kubectl apply -f deploy/hpa-tuned.yaml
# Wait 100 seconds
kubectl -n shop get hpa
```

### Step 10: Send the same spike

Same traffic, same pods.

```bash
kubectl -n shop set env deployment/customers RPS=40
# Wait 100 seconds
kubectl -n shop get hpa
./scripts/verdict.sh 60
```

### Step 11: Clean up

```bash
kind delete cluster --name lab --kubeconfig .kubeconfig
```

## Note on the verdict window

The verdict checks run after the first seconds of each spike, so they can pass while the dashboard shows customers giving up. Use Grafana, or run the verdict with a window that covers the start of the spike.

## Challenges

Predict the result before each run.

1. Set `minReplicas: 6` and measure from the moment the spike starts.
2. Remove the CPU limit. What changes?
3. Find the lowest minReplicas that keeps the SLO.
4. Double `CPU_WORK` and predict the replica count first.

## Safety

The lab never changes your default Kubernetes context. Grafana allows anonymous viewing and its ports are for local use. Do not expose them.

## License

MIT
