# README-nodes-ubuntu.md

**Raspberry Pi (ARM64) & Ubuntu Server/Ubuntu Core nodes + k3s (mixed roles)**

This guide covers **Raspberry Pis** and any **Ubuntu Server (x86/ARM)** nodes that will join your homelab k3s cluster, as either **control‑plane** or **worker** nodes.

> Cluster facts
>
> - LAN: `10.68.69.0/24`  ·  DHCP: `10.68.69.100-199`  ·  VIP: `10.68.69.208`
> - MetalLB: `10.68.69.50-10.68.69.99`  ·  Domain: `ekenstam.dev`
> - GitOps: Argo CD repo `https://github.com/tekenstam/k8s-homelab`

---

## 0) OS choices
- **Ubuntu Server 22.04/24.04 (ARM64)** for Raspberry Pi 4/5 — recommended.
- **Ubuntu Core 24** (immutable, snap‑based) — optional; start with Ubuntu Server if new.

Avoid microSD for storage nodes. Prefer **USB SSDs** for Pis.

---

## 1) Flash & first boot (Raspberry Pi)
1. Use **Raspberry Pi Imager** → Ubuntu Server 22.04/24.04 (ARM64).
2. Preconfigure hostname, user, SSH, and **static IP** outside DHCP (10.68.69.20–49) or reserve via DHCP.
3. Boot with SSD attached (USB3).

Baseline:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl git jq nfs-common open-iscsi apparmor-utils
sudo systemctl enable --now iscsid
# Optional on ARM:
sudo apt install -y linux-modules-extra-$(uname -r)
```

---

## 2) Prepare disk for Longhorn
```bash
sudo mkfs.ext4 -F /dev/sda   # DANGER: confirm device
sudo mkdir -p /var/lib/longhorn
echo "/dev/sda /var/lib/longhorn ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo mount -a
```

---

## 3) Optional: Tailscale per node
```bash
curl -fsSL https://tailscale.com/install.sh | sudo bash
sudo tailscale up --authkey tskey-... --hostname $(hostname) --ssh
```

---

## 4) k3s install — mixed roles
Choose a **strong shared token** and keep it safe.

### 4.1 First control‑plane (cluster‑init)
```bash
export K3S_TOKEN="<STRONG-SHARED-TOKEN>"
export K3S_KUBECONFIG_MODE="644"
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init --disable traefik --tls-san 10.68.69.208 --node-name $(hostname)" sh -
```

### 4.2 Additional control‑planes
```bash
export K3S_TOKEN="<STRONG-SHARED-TOKEN>"
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik --server https://10.68.69.208:6443 --tls-san 10.68.69.208 --node-name $(hostname)" sh -
```

### 4.3 Workers
```bash
export K3S_TOKEN="<STRONG-SHARED-TOKEN>"
export K3S_URL="https://10.68.69.208:6443"
curl -sfL https://get.k3s.io | K3S_URL=${K3S_URL} K3S_TOKEN=${K3S_TOKEN} INSTALL_K3S_EXEC="agent --node-name $(hostname)" sh -
```

---

## 5) kube‑vip & GitOps bootstrap
```bash
kubectl apply -f bootstrap/kube-vip.yaml
kubectl apply -f clusters/onprem/apps.yaml
kubectl apply -f appsets/shared-infra-appset.yaml
```

---

## 6) Node roles, labels & taints
```bash
kubectl label node pi-01 role=control-plane
kubectl label node pi-w01 role=worker
kubectl label node vm-03 role=ingress
kubectl label node pi-w02 storage=ssd
kubectl taint node vm-03 node-role=ingress:NoSchedule
```

---

## 7) Upgrades & troubleshooting
- **Ubuntu**: `sudo apt full-upgrade -y && sudo reboot`
- **k3s**: drain/cordon servers one-by-one, then workers; pin channel with `INSTALL_K3S_CHANNEL=stable`.
- Check: `sudo journalctl -u k3s* -b` for errors; verify `/var/lib/longhorn` and `iscsid`.
