# siro-red: Istio Ambient Mesh via Argo CD App of Apps

This repository bootstraps Istio Ambient (v1.31.3) on the user's Debian 13 / kubeadm / containerd / Flannel single-node cluster, **after** Argo CD is already installed.

## Layout

```text
siro-red-gitops/
├── bootstrap/
│   └── istio-ambient-parent.yaml       # bootstrap this once using kubectl
└── applications/
    └── istio/
        ├── 01-istio-base.yaml          # child, manual sync
        ├── 02-istiod.yaml              # child, manual sync
        ├── 03-istio-cni.yaml           # child, manual sync
        └── 04-ztunnel.yaml             # child, manual sync
```

## Why manual sync?

The parent Application automatically registers 4 child Applications. The child Applications deliberately **do not** have `syncPolicy.automated`, to guarantee an operator can verify Istio's required installation order. Argo CD sync waves alone don't guarantee lower-wave child Apps have completed reconciliation (Application health isn't assessed by default). You can enable child automatic sync **after** verifying the initial installation and adding robust dependency/health-gating policies.

## Prerequisites

- `kubectl get nodes` = Ready
- `kubectl get pods -n argocd` = ready
- `kubectl get pods -n kube-flannel` = running
- `kubectl get applications -n argocd` does not show an existing instance of these four apps or other Istio installations
- Check `sudo ls -l /etc/cni/net.d/` and `/opt/cni/bin`; make sure paths in `03-istio-cni.yaml` match your containerd settings.
- Back up your CNI config on the server before installing `istio-cni`:

```bash
sudo tar -czf /root/cni-before-istio.tar.gz -C /etc/cni net.d
```

## 1. Publish Git repository

Create an EMPTY public GitHub repository `siro-red-gitops` (or use a private one, but configure its credentials in Argo CD). Limit who can push: App of Apps is an admin-only configuration.

After unzipping, open `bootstrap/istio-ambient-parent.yaml` and replace `REPLACE_ME` with your actual GitHub username/org. On the Mac terminal:

```bash
cd ~/Downloads/siro-red-istio-ambient-appofapps
# edit bootstrap/istio-ambient-parent.yaml
# rename top-level directory if you want; its name does not affect the manifests' paths

git init
git branch -M main
git add .
git commit -m 'Bootstrap Istio Ambient with Argo CD App of Apps'
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/siro-red-gitops.git
git push -u origin main
```

## 2. Parent bootstrap

On Mac, copy one parent manifest to FIREBAT:

```bash
scp bootstrap/istio-ambient-parent.yaml debian@192.168.0.25:~/istio-ambient-parent.yaml
ssh debian@192.168.0.25
```

On FIREBAT:

```bash
kubectl create namespace istio-system --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f ~/istio-ambient-parent.yaml
kubectl -n argocd get applications
```

The parent should create four child apps, initially `OutOfSync`.

## 3. Synchronize children one by one in Argo CD UI

1. `istio-base`: click Sync; verify `Synced`, CRDs established:
   `kubectl wait --for=condition=Established crd/virtualservices.networking.istio.io --timeout=300s`
2. `istiod`: Sync; verify:
   `kubectl -n istio-system rollout status deploy/istiod --timeout=300s`
3. `istio-cni`: Sync **only after** earlier steps healthy; verify:
   `kubectl -n istio-system rollout status ds/istio-cni-node --timeout=300s`
   `kubectl get pods -n kube-flannel && kubectl get nodes` 
4. `ztunnel`: Sync; verify:
   `kubectl -n istio-system rollout status ds/ztunnel --timeout=300s`

Check:
```bash
kubectl -n argocd get applications
kubectl -n istio-system get pods -o wide
kubectl get nodes
kubectl get pods -n kube-flannel
```

Do not delete or replace Flannel. `istio-cni` chains atop the existing network. Don't enable ambient mode on `default`, `argocd`, `kube-system` during initial testing.

## 4. Isolated test

```bash
kubectl create namespace ambient-demo
kubectl label namespace ambient-demo istio.io/dataplane-mode=ambient
kubectl -n ambient-demo create deployment nginx --image=nginx:stable-alpine
kubectl -n ambient-demo expose deployment nginx --port=80 --target-port=80
kubectl -n ambient-demo rollout status deploy/nginx
kubectl -n ambient-demo run client --image=curlimages/curl:8.16.0 --restart=Never --command -- sleep 3600
kubectl -n ambient-demo wait --for=condition=Ready pod/client --timeout=180s
kubectl -n ambient-demo exec pod/client -- curl -I http://nginx
kubectl -n ambient-demo get pods -o wide
```

`curl` working and 1/1 containers proves the app is reachable without an injected sidecar. It **does not** alone prove mTLS. Use `istioctl ztunnel-config workloads` (with Istio 1.31.3 istioctl) to inspect PROTOCOL=HBONE for the mesh-enrolled pods, and confirm traffic metrics/logs when verifying actual mTLS behavior.

## Notes

- Istio Helm charts are rendered via Helm inside Argo CD, but **Argo CD**, not Helm CLI, manages lifecycle. Therefore `helm ls` is not the reliable status check.
- Versions are deliberately pinned to Istio v1.31.3 across all four Applications.
- `istiod` Helm values include `base.validationFailurePolicy: Fail` to avoid GitOps/Server-Side-Apply validation-webhook field-manager conflicts.
- Gateway API CRDs and waypoint proxy are optional next steps and are intentionally not deployed by this starter.
- Private GitHub repositories require adding repository credentials in Argo CD settings before bootstrapping.
- Publish no credentials or kubeconfig secrets. Protect write access to the repository.

## References

- https://istio.io/latest/docs/ambient/install/helm/
- https://istio.io/latest/docs/ambient/install/platform-prerequisites/
- https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
- https://argo-cd.readthedocs.io/en/stable/operator-manual/health/
- https://istio.io/latest/docs/ambient/usage/verify-mtls-enabled/
