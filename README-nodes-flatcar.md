# README-nodes-flatcar.md

**Container‑native Proxmox VMs with Flatcar Container Linux + k3s (mixed roles)**

This guide shows how to provision **Flatcar Container Linux** VMs on **Proxmox** as either **control‑plane** or **worker** nodes for your homelab k3s cluster.

> Cluster facts
>
> - LAN: `10.68.69.0/24`
> - DHCP: `10.68.69.100-199`
> - MetalLB: `10.68.69.50-10.68.69.99`
> - Control‑plane VIP (kube‑vip): `10.68.69.208`
> - Domain: `ekenstam.dev`
> - GitOps: Argo CD (apps sync from `https://github.com/tekenstam/k8s-homelab`)

---

## 0) Overview: mixed‑role layout
You can run **any mix** of control‑planes and workers on Pis and/or Proxmox VMs. Ensure you have **≥3 control‑plane nodes** total for HA (across any platform).

**Recommended labels/taints** (apply after join):

```bash
kubectl label node vm-01 role=control-plane
kubectl label node vm-02 role=worker
kubectl label node vm-03 role=ingress
kubectl label node vm-02 storage=ssd
kubectl taint node vm-03 node-role=ingress:NoSchedule
```

Longhorn expects **open-iscsi** on nodes that host volumes. On Flatcar, use a dedicated disk mounted to `/var/lib/longhorn`.

---

## 1) Prepare Proxmox template
1. **Download** Flatcar CL ISO (stable) and upload to Proxmox ISO storage.
2. Create a **cloud‑init** ready VM template:
   - Disk: 20–40 GiB (more if running Longhorn replicas on this VM)
   - CPU/RAM: 2 vCPU / 4–8 GiB RAM
   - NIC: VirtIO bridged to your LAN
   - Add **Cloud‑Init** drive
3. Convert to a **template** so you can clone per role.

> Tip: Give **static IPs outside DHCP** (e.g., 10.68.69.20–49 for VMs). Proxmox cloud‑init can set it, or use DHCP reservations.

---

## 2) Author Ignition with **Butane** (per role)
Install Butane:
```bash
brew install butane   # macOS
# Linux: https://github.com/coreos/butane/releases
```

### 2.1 Control‑plane VM (first server / cluster‑init)
`flatcar-k3s-cp1.bu`:
```yaml
variant: flatcar
version: 1.0.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-ed25519 AAAA...yourkey
storage:
  files:
    - path: /etc/environment
      mode: 0644
      contents:
        inline: |
          K3S_TOKEN="<REPLACE_WITH_STRONG_SHARED_TOKEN>"
          K3S_URL="https://10.68.69.208:6443"
    - path: /etc/systemd/system/k3s-install.service
      mode: 0644
      contents:
        inline: |
          [Unit]
          Description=Install k3s (server, cluster-init)
          After=network-online.target
          Wants=network-online.target
          [Service]
          Type=oneshot
          EnvironmentFile=/etc/environment
          ExecStart=/usr/bin/bash -c 'curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init --disable traefik --tls-san 10.68.69.208 --node-name %H" sh -'
          [Install]
          WantedBy=multi-user.target
systemd:
  units:
    - name: k3s-install.service
      enabled: true
```
Compile:
```bash
butane --strict -o flatcar-k3s-cp1.ign flatcar-k3s-cp1.bu
```

### 2.2 Additional control‑plane VMs
`flatcar-k3s-cpN.bu`:
```yaml
variant: flatcar
version: 1.0.0
storage:
  files:
    - path: /etc/environment
      mode: 0644
      contents:
        inline: |
          K3S_TOKEN="<SAME_TOKEN_AS_CP1>"
          K3S_URL="https://10.68.69.208:6443"
    - path: /etc/systemd/system/k3s-install.service
      mode: 0644
      contents:
        inline: |
          [Unit]
          Description=Install k3s (server, join existing)
          After=network-online.target
          Wants=network-online.target
          [Service]
          Type=oneshot
          EnvironmentFile=/etc/environment
          ExecStart=/usr/bin/bash -c 'curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik --server ${K3S_URL} --tls-san 10.68.69.208 --node-name %H" sh -'
          [Install]
          WantedBy=multi-user.target
systemd:
  units:
    - name: k3s-install.service
      enabled: true
```

### 2.3 Worker VMs
`flatcar-k3s-worker.bu`:
```yaml
variant: flatcar
version: 1.0.0
storage:
  files:
    - path: /etc/environment
      mode: 0644
      contents:
        inline: |
          K3S_TOKEN="<SAME_TOKEN_AS_CP1>"
          K3S_URL="https://10.68.69.208:6443"
    - path: /etc/systemd/system/k3s-install.service
      mode: 0644
      contents:
        inline: |
          [Unit]
          Description=Install k3s (agent)
          After=network-online.target
          Wants=network-online.target
          [Service]
          Type=oneshot
          EnvironmentFile=/etc/environment
          ExecStart=/usr/bin/bash -c 'curl -sfL https://get.k3s.io | K3S_URL=${K3S_URL} K3S_TOKEN=${K3S_TOKEN} INSTALL_K3S_EXEC="agent --node-name %H" sh -'
          [Install]
          WantedBy=multi-user.target
systemd:
  units:
    - name: k3s-install.service
      enabled: true
```
Compile each with `butane --strict -o <name>.ign <name>.bu`.

---

## 3) Provision on Proxmox
1. Clone from the template.
2. Attach Ignition via Cloud‑Init → Custom files.
3. Set hostname, static IP, DNS.
4. Boot; the VM installs & joins automatically.

> Join token (if needed): `/var/lib/rancher/k3s/server/node-token`

---

## 4) Longhorn on Flatcar VMs
```bash
sudo mkfs.ext4 -F /dev/vdb
sudo mkdir -p /var/lib/longhorn
echo "/dev/vdb /var/lib/longhorn ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo mount -a
kubectl label node $(hostname) storage=ssd
```

---

## 5) kube‑vip & GitOps bootstrap
```bash
kubectl apply -f bootstrap/kube-vip.yaml
kubectl apply -f clusters/onprem/apps.yaml
kubectl apply -f appsets/shared-infra-appset.yaml
```

---

## 6) Optional: Tailscale on Flatcar
```bash
curl -fsSL https://tailscale.com/install.sh | sudo bash
sudo tailscale up --authkey tskey-... --hostname $(hostname)
```

---

## 7) Maintenance
- Flatcar updates via update‑engine (reboot applies).
- k3s upgrades in rolling fashion (cordon/drain control‑planes first).
