# Metrics Server Outage - Root Cause Analysis

**Date:** 2026-03-18
**Cluster:** deepak-kind-cluster (Kind, Kubernetes v1.33.1)
**Impact:** `kubectl top` commands failing with `Metrics API not available`

---

## Issue Description

The Kubernetes metrics-server was deployed and the pod was in `Running` state, but the Metrics API was unavailable. Any command relying on the metrics API (`kubectl top nodes`, `kubectl top pods`, HPA scaling) failed with the error:

```
error: Metrics API not available
```

The APIService `v1beta1.metrics.k8s.io` reported `FailedDiscoveryCheck`:

```
failing or missing response from https://10.96.85.36:443/apis/metrics.k8s.io/v1beta1:
Get "https://10.96.85.36:443/apis/metrics.k8s.io/v1beta1": net/http: request canceled
while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

---

## RCA Steps

### Step 1: Verify metrics-server pod status

```bash
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get deployment metrics-server -n kube-system -o wide
```

**Finding:** Pod was `Running` (1/1 Ready), deployment was available. No obvious issue at the pod level.

### Step 2: Test the Metrics API

```bash
kubectl top nodes
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

**Finding:** `kubectl top` returned `Metrics API not available`. The APIService showed `status: "False"` with reason `FailedDiscoveryCheck` — the kube-apiserver could not reach the metrics-server Service ClusterIP (`10.96.85.36:443`). Connection was timing out.

### Step 3: Check Service and Endpoints

```bash
kubectl get svc metrics-server -n kube-system -o yaml
kubectl get endpoints metrics-server -n kube-system -o yaml
```

**Finding:** Service configuration was correct (port 443 -> targetPort 10250). Endpoints were populated with the correct pod IP. The Service itself was not the issue.

### Step 4: Check node health

```bash
kubectl get nodes -o wide
kubectl describe node deepak-kind-cluster-control-plane
```

**Finding:** The control-plane node was in `NotReady` state. All conditions showed `NodeStatusUnknown` with message `Kubelet stopped posting node status`. Last heartbeat was weeks ago. Both nodes were showing the same Internal IP (`172.19.0.3`), which was abnormal.

### Step 5: Check Docker container IPs vs configured IPs

```bash
docker inspect deepak-kind-cluster-control-plane -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker inspect deepak-kind-cluster-worker -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

**Finding:** IPs had swapped after a Docker restart:

| Node | Original IP | Current IP |
|------|------------|------------|
| control-plane | 172.19.0.3 | **172.19.0.2** |
| worker | 172.19.0.2 | **172.19.0.3** |

### Step 6: Check kubelet and kube-apiserver configuration

```bash
docker exec deepak-kind-cluster-control-plane grep server /etc/kubernetes/kubelet.conf
docker exec deepak-kind-cluster-control-plane grep advertise /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Finding:** The kube-apiserver manifest was updated to `--advertise-address=172.19.0.2` (correct), but `kubelet.conf` still had `server: https://172.19.0.3:6443` (stale IP). The kubelet could not reach the API server.

### Step 7: Confirm kubelet logs

```bash
docker exec deepak-kind-cluster-control-plane journalctl -u kubelet --no-pager -n 30
```

**Finding:** Kubelet was continuously failing with:

```
dial tcp 172.19.0.3:6443: connect: connection refused
```

---

## Root Cause

After a Docker/machine restart, the Kind cluster Docker containers were assigned swapped IP addresses. The control-plane moved from `172.19.0.3` to `172.19.0.2` and the worker moved from `172.19.0.2` to `172.19.0.3`.

While most kubeconfig files were updated by kubeadm, the **control-plane kubelet's kubeconfig** (`/etc/kubernetes/kubelet.conf`) retained the stale IP `172.19.0.3`, causing the kubelet to fail connecting to the API server.

**Chain of failure:**

```
IP swap after Docker restart
  → kubelet.conf has stale API server IP
    → control-plane kubelet can't register with API server
      → control-plane node marked NotReady
        → kube-proxy / networking broken on control-plane
          → kube-apiserver can't reach metrics-server ClusterIP
            → Metrics API unavailable
              → kubectl top fails
```

---

## Resolution

1. Updated the stale IP in kubelet.conf:

```bash
docker exec deepak-kind-cluster-control-plane \
  sed -i 's|server: https://172.19.0.3:6443|server: https://172.19.0.2:6443|' \
  /etc/kubernetes/kubelet.conf
```

2. Restarted the kubelet:

```bash
docker exec deepak-kind-cluster-control-plane systemctl restart kubelet
```

3. Verified the fix:

```bash
kubectl get nodes                                    # Both nodes Ready
kubectl get apiservice v1beta1.metrics.k8s.io        # AVAILABLE: True
kubectl top nodes                                    # Metrics returned successfully
```

---

## Mitigation Steps for Future

### 1. Use hostnames instead of hardcoded IPs in kubeconfigs

The worker node's kubelet.conf uses the hostname (`deepak-kind-cluster-control-plane`) instead of an IP, which is why it was unaffected. Update the control-plane's kubelet.conf to do the same:

```bash
docker exec deepak-kind-cluster-control-plane \
  sed -i 's|server: https://172.19.0.2:6443|server: https://deepak-kind-cluster-control-plane:6443|' \
  /etc/kubernetes/kubelet.conf

docker exec deepak-kind-cluster-control-plane systemctl restart kubelet
```

### 2. Use fixed IPs for Kind containers

Configure static IPs in your Kind cluster config to prevent IP swaps on restart:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
nodes:
  - role: control-plane
  - role: worker
```

Alternatively, use a Docker network with fixed IPs:

```bash
docker network create --subnet=172.19.0.0/16 kind
# Then assign specific IPs when creating the Kind cluster
```

### 3. Add a post-restart health check script

Create a script to run after Docker/machine restarts:

```bash
#!/bin/bash
# check-kind-cluster.sh

CLUSTER_NAME="deepak-kind-cluster"
CP_CONTAINER="${CLUSTER_NAME}-control-plane"

# Get current container IP
CURRENT_IP=$(docker inspect "$CP_CONTAINER" -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')

# Get configured API server IP in kubelet.conf
CONFIGURED_IP=$(docker exec "$CP_CONTAINER" grep server /etc/kubernetes/kubelet.conf | grep -oP 'https://\K[^:]+')

if [ "$CURRENT_IP" != "$CONFIGURED_IP" ]; then
  echo "IP mismatch detected: container=$CURRENT_IP, kubelet.conf=$CONFIGURED_IP"
  echo "Fixing kubelet.conf..."
  docker exec "$CP_CONTAINER" sed -i "s|server: https://${CONFIGURED_IP}:6443|server: https://${CURRENT_IP}:6443|" /etc/kubernetes/kubelet.conf
  docker exec "$CP_CONTAINER" systemctl restart kubelet
  echo "Fixed. Waiting for node to become Ready..."
  sleep 15
  kubectl get nodes
else
  echo "IPs match ($CURRENT_IP). No fix needed."
fi
```

### 4. Monitor node readiness

Set up alerts or periodic checks for node readiness so this class of issue is caught early:

```bash
# Quick check - add to a cron or startup script
kubectl get nodes --no-headers | grep -v " Ready " && echo "WARNING: Not all nodes are Ready"
```
