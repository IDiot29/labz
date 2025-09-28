# Cilium Bootstrap Failure in K3s Cluster

## Issue Summary
- **Symptoms**: Every `cilium` DaemonSet pod stuck in `Init:CrashLoopBackOff`; many workloads reporting `Unknown` state across namespaces.
- **Root Cause**: Cilium runs with kube-proxy replacement but cannot reach the Kubernetes API via the service VIP (`10.43.0.1:443`). Until Cilium programs the service IP, the VIP is unroutable, creating a circular dependency during startup.

## Temporary Remediation (Applied)
1. Patch the `cilium-config` ConfigMap to point directly at a reachable control-plane endpoint:
   ```bash
   kubectl patch configmap cilium-config -n kube-system \
     --type merge \
     -p '{"data":{"k8s-service-host":"10.10.0.26","k8s-service-port":"6443"}}'
   ```
2. Override the Kubernetes service environment variables on the Cilium workloads so init containers immediately use the node IP:
   ```bash
   kubectl set env daemonset cilium -n kube-system \
     KUBERNETES_SERVICE_HOST=10.10.0.26 \
     KUBERNETES_SERVICE_PORT=6443

   kubectl set env deployment cilium-operator -n kube-system \
     KUBERNETES_SERVICE_HOST=10.10.0.26 \
     KUBERNETES_SERVICE_PORT=6443
   ```
3. Restart the DaemonSet and operator to consume the new settings:
   ```bash
   kubectl rollout restart daemonset cilium -n kube-system
   kubectl rollout restart deployment cilium-operator -n kube-system
   ```
4. Verify recovery:
   ```bash
   kubectl get pods -n kube-system -l k8s-app=cilium
   kubectl get pods -A
   ```

## Permanent Fix Options
### 1. Front Cilium With a Control-Plane Load Balancer (Recommended)
- Provision a stable VIP that fronts all masters on TCP `6443` (hardware LB, cloud LB, etc.).
- Update Helm/cilium-cli values or `cilium-config` so `k8s-service-host` points to the VIP.
- Update kubeconfigs and node bootstrap configs to use the same VIP for consistency.

### 2. Keepalived + HAProxy (Bare Metal)
- Install `keepalived` and `haproxy` on each master.
- Use Keepalived VRRP to advertise a floating IP (e.g., `10.10.0.200`).
- Configure HAProxy to balance `*:6443` to all masters:
  ```conf
  frontend kubernetes
    bind 10.10.0.200:6443
    mode tcp
    default_backend kubernetes-masters

  backend kubernetes-masters
    mode tcp
    balance roundrobin
    server master01 10.10.0.26:6443 check
    server master02 10.10.0.133:6443 check
    server master03 10.10.0.7:6443 check
  ```
- Point Cilium and kube clients at `10.10.0.200:6443`.

### 3. kube-vip (DaemonSet-Based VIP)
- Deploy kube-vip in `kube-system` to manage a virtual IP and load-balance TCP `6443`.
- Suitable for kubeadm, k3s, and other bare-metal installs; integrates directly with Kubernetes lifecycle.
- Configure Cilium Helm values to use the kube-vip address.

### 4. MetalLB Backed LoadBalancer Service
- If MetalLB is present, expose the control-plane via a `LoadBalancer` Service that announces a VIP.
- Ensure health checks target `/healthz` or `/readyz` on the API.
- Update Cilium (`k8sServiceHost`/`k8sServicePort`) and kubeconfigs to use that Service IP.

### 5. Re-enable kube-proxy (Fallback)
- If load balancer options are unavailable, run kube-proxy or set Cilium `kubeProxyReplacement=partial`.
- This restores service VIP programming during bootstrap but sacrifices some performance features.

## Helm / cilium-cli Persistence
- Add the chosen host/port to your Helm values:
  ```yaml
  k8sServiceHost: 10.10.0.200
  k8sServicePort: 6443
  ```
- Re-run `cilium upgrade` or `helm upgrade` so future upgrades retain the configuration.
- Remove ad-hoc `kubectl set env` overrides once the chart values are authoritative.

## Validation Checklist
- `kubectl get pods -A` shows all Cilium pods `1/1 Running` and workloads healthy.
- `kubectl exec -n kube-system deploy/coredns -- dig kubernetes.default.svc.cluster.local` succeeds.
- API server reachable from nodes: `curl -k https://<VIP>:6443/healthz`.
- No `Init:CrashLoopBackOff` or `Unknown` pods remain across namespaces.

