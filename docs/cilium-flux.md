# Cilium Deployment via Flux

## Layout
- `helm/cilium/repository.yaml` — Flux `HelmRepository` for https://helm.cilium.io (namespace `flux-system`).
- `infrastructure/cilium/` — Kustomize overlay used by Flux:
  - `values.yaml` — Helm values capturing kube-proxy replacement, routing mode, IPAM range, and API VIP (`10.10.0.95:6443`).
  - `kustomization.yaml` — Generates `ConfigMap/cilium-values` (hash disabled) from `values.yaml` and applies the HelmRelease.
  - `release.yaml` — Flux `HelmRelease` targeting chart version `1.18.2` in namespace `kube-system`.

## Apply Manually (bootstrap)
```bash
kubectl apply -f helm/cilium/repository.yaml
kubectl apply -k infrastructure/cilium
```

## Notes
- The Helm release name remains `cilium`; uninstall any existing CLI-installed release first (`helm uninstall cilium -n kube-system`) or let Flux adopt it after removing the old Helm secret.
- Values keep `kubeProxyReplacement=strict`, `routingMode=tunnel`, and the pod/service CIDRs set to match the cluster (`10.42.0.0/16`, `10.43.0.0/16`).
- HAProxy load-balancer endpoint (`10.10.0.95:6443`) is wired through `k8sServiceHost`/`k8sServicePort` so agents can talk to the API during bootstrap.
