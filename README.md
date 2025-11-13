# Homelab GitOps (Argo CD + k3s + MetalLB + Longhorn + NGINX + Cloudflare + SOPS)
Domain: **ekenstam.dev**  
LAN: **10.68.69.0/24**  
DHCP: **10.68.69.100-199**  
MetalLB pool: **10.68.69.50-10.68.69.99**  
Control-plane VIP (kube-vip): **10.68.69.208**  
TrueNAS SCALE (S3): **nas-01.ekenstam.dev**  
Repo: **https://github.com/tekenstam/k8s-homelab**

## Bootstrap
1) **kube-vip** VIP:
```bash
kubectl apply -f bootstrap/kube-vip.yaml
```
2) **Argo CD** (upstream install):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
3) **App-of-Apps + ApplicationSet**:
```bash
kubectl apply -f clusters/onprem/apps.yaml
kubectl apply -f appsets/shared-infra-appset.yaml
```
> Argo will reconcile MetalLB, Longhorn, ingress, cert-manager, external-dns, Argo (self-managed), and secrets via ksops.

---

## Cloudflare integration
- **cert-manager** DNS-01 (Let’s Encrypt) + **external-dns** (provider=cloudflare)
- Create a Cloudflare API token with: Zone:DNS **Edit**, Zone:Zone **Read**.
- Put token in `secrets/cloudflare-api-token.yaml` (ns `cert-manager`) and `secrets/external-dns-cloudflare.yaml` (ns `external-dns`) — encrypt with SOPS.

---

## SOPS + age with KSOPS in Argo CD
1) Generate an **age** key:
```bash
age-keygen -o age.agekey
```
2) Create in-cluster Secret (do **not** commit the key):
```bash
kubectl -n argocd create secret generic sops-age --from-file=keys.txt=age.agekey
```
3) Encrypt secrets (replace with your age public key):
```bash
export AGE_RECIPIENT="$(age-keygen -y age.agekey)"
sops --encrypt --age "$AGE_RECIPIENT" -i secrets/longhorn-s3-cred.yaml
sops --encrypt --age "$AGE_RECIPIENT" -i secrets/cloudflare-api-token.yaml
sops --encrypt --age "$AGE_RECIPIENT" -i secrets/external-dns-cloudflare.yaml
```
4) Argo Application **`secrets-onprem`** uses **ksops** to decrypt and apply at sync.

Safety rails: `.sops.yaml`, `.gitignore`, pre-commit hooks and CI are included.

---

## Multi-cluster
When you add EKS via `argocd cluster add`, the `ApplicationSet` deploys shared infra there too. Use cluster overlays under `clusters/<name>/` as needed.
