# KubeVirt SR-IOV Validation on IBM Power (ppc64le)

---

## 1. Objective

This document records the work performed to evaluate and validate SR-IOV device passthrough for KubeVirt VMs on IBM Power (ppc64le architecture), as defined in Jira ticket POWEROSV-123.

The expected integration stack under test:

```
IBM Power Hardware (Power10, ppc64le)
    └── PowerVM Hypervisor (HMC-managed)
            └── CentOS 10 Stream LPAR
                    └── kubeadm Kubernetes Cluster
                            ├── Calico CNI
                            ├── Multus CNI
                            ├── SR-IOV Network Operator
                            └── KubeVirt
                                    └── VirtualMachine (SR-IOV VF passthrough)
```

**Goals:**
- Deploy KubeVirt on a single-node kubeadm cluster on IBM Power
- Install and validate SR-IOV Network Operator integration with Multus and KubeVirt
- Verify SR-IOV Virtual Function (Logical Port) allocation and VM attachment
- Document architecture-specific compatibility findings for ppc64le

**Note:** Per guidance from the ticket assignee, this is a research/discovery task. A finding of "does not work" with documented root cause is a valid and accepted outcome.

---

## 2. Environment

### 2.1 Hardware

| Property | Value |
|---|---|
| Architecture | IBM Power (ppc64le) |
| Hypervisor | PowerVM (PHYP) |
| System | IBM Power10 |
| LPAR hostname | `onprem133725233.pokprv.stglabs.ibm.com` |
| CPUs | 8 |
| Memory | 30 GiB |
| OS | CentOS Stream 10 |
| Swap | Disabled |

### 2.2 Network Interfaces

| Interface | Driver | Type | PCI Address |
|---|---|---|---|
| `env32` | `ibmveth` | IBM Virtual Ethernet (VIOS/SEA) | `/vdevice/l-lan@30000020` |
| `enP32770p1s0` | `mlx5_core` | SR-IOV Logical Port (HMC-assigned) | `8002:01:00.0` |

**SR-IOV Logical Port details (`enP32770p1s0`):**
```
Driver:      mlx5_core
Vendor:      Mellanox Technologies (15b3)
Device ID:   1016 (MT27710 Family ConnectX-4 Lx Virtual Function)
PCI Address: 8002:01:00.0
IOMMU Group: 0 (vfio_iommu_spapr_tce)
Adapter:     U78DA.ND0.WZS04W2-P0-C7 — IBM PCIe3 2-Port 25/10 Gb NIC&RoCE SFP28
HMC Config:  Capacity 2%, Promiscuous OFF, MAC Allow All, VLAN Allow All
```

### 2.3 Software Versions

| Component | Version |
|---|---|
| CentOS Stream | 10 (Coughlan) |
| Kernel | 6.12.0-257.el10.ppc64le |
| Go | 1.26.5 (Red Hat 1.26.5-1.el10) |
| Kubernetes (kubeadm) | v1.33.x |
| kubectl | v1.36.3 |
| KubeVirt | v1.0.7-ppc64le (quay.io/pkenchap) |
| Multus CNI | snapshot-thick (ghcr.io/k8snetworkplumbingwg) |
| Calico | v3.x |
| SR-IOV Network Operator | v1.6.0 |
| kustomize | v5.3.0 |
| skopeo | 1.24.0 |

---

## 3. Work Completed — Chronological Timeline

### 3.1 Kubernetes Cluster Installation (kubeadm)

A single-node kubeadm cluster was deployed directly on the IBM Power LPAR. KIND (Kubernetes in Docker) was initially used for KubeVirt testing but was not suitable for SR-IOV testing because KIND runs Kubernetes inside Docker containers, isolating it from host hardware and making SR-IOV VFs invisible to the cluster.

**Steps performed:**
- Disabled swap (`swapoff -a`)
- Loaded kernel modules: `overlay`, `br_netfilter`
- Set kernel parameters for bridged network traffic
- Installed `containerd` with `SystemdCgroup=true`
- Added Kubernetes v1.33 yum repo
- Installed `kubelet`, `kubeadm`, `kubectl`
- Initialized cluster: `kubeadm init --pod-network-cidr=10.244.0.0/16`
- Removed control-plane taint for single-node scheduling

---

### 3.2 Calico CNI — iptables Module Issue

**Problem observed:**  
After kubeadm init and Calico manifest application, Calico pods entered `CrashLoopBackOff`. The node remained in `NotReady` state.

**Investigation:**
```bash
lsmod | grep ip_tables    # returned empty — modules not loaded
iptables-save             # failed with kernel module error
```

**Root cause:**  
CentOS Stream 10 does not autoload `ip_tables`, `iptable_filter`, and `iptable_nat` kernel modules at boot. Calico requires these for network policy enforcement.

**Resolution:**
```bash
modprobe ip_tables
modprobe iptable_filter
modprobe iptable_nat

# Persist across reboots
cat <<EOF > /etc/modules-load.d/iptables.conf
ip_tables
iptable_filter
iptable_nat
EOF
```

After loading the modules, Calico pods became `Running` and the node reached `Ready` state.

---

### 3.3 Docker Hub Rate Limiting (ImagePullBackOff)

During initial cluster validation using `nginx` test pods, `ImagePullBackOff` errors were observed due to Docker Hub pull rate limits (HTTP 429). This is expected behaviour on IBM network egress to Docker Hub. Subsequent tests used images from `quay.io` and `ghcr.io` registries without issue.

---

### 3.4 KubeVirt Installation and Verification

KubeVirt was installed using ppc64le-specific images from `quay.io/pkenchap` (IBM Power fork maintained by the team):

**Operator YAML:**  
Downloaded official KubeVirt operator manifest (v1.8.4) and patched all image references to `quay.io/pkenchap/virt-operator:v1.0.7-ppc64le`.

**KubeVirt CR:**
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  imageRegistry: 'quay.io/pkenchap'
  configuration:
    developerConfiguration:
      featureGates:
        - Root
  imagePullPolicy: Always
```

**RBAC gap encountered:**  
The operator failed with:
```
clusterroles.rbac.authorization.k8s.io "kubevirt.io:admin" is forbidden:
attempting to grant RBAC permissions not currently held:
  {APIGroups: ["plugin.kubevirt.io"], Resources: ["plugins"]}
  {APIGroups: ["subresources.kubevirt.io"], Resources: ["virtualmachines/removememorydump"]}
```
**Resolution:** Manually patched the `kubevirt-operator` ClusterRole to add the two missing permissions. This is a known version mismatch gap between the official v1.8.4 operator YAML and the ppc64le v1.0.7 images.

**Verification:**
```bash
kubectl -n kubevirt get kubevirt kubevirt -o jsonpath='{.status.phase}'
# Output: Deployed

kubectl -n kubevirt get pods
# virt-api, virt-controller, virt-handler, virt-operator all Running
```

**Test VM deployed and verified:**
```bash
kubectl get vmi testvmfedora
# NAME           PHASE     IP            NODENAME
# testvmfedora   Running   10.244.0.15   onprem133725233...

virtctl console testvmfedora
# Successfully logged in to Fedora 39 Cloud Edition
```

---

### 3.5 Multus CNI Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```

**Verification:**
```bash
kubectl -n kube-system get pods -l app=multus
# kube-multus-ds-5mqlq   1/1   Running

kubectl -n kube-system logs kube-multus-ds-5mqlq --tail=5
# Generated MultusCNI config wrapping Calico as primary CNI
# Started file watcher on /host/etc/cni/net.d/10-calico.conflist
```

Image used: `ghcr.io/k8snetworkplumbingwg/multus-cni:snapshot-thick` — confirmed ppc64le support via podman manifest inspect.

---

### 3.6 HMC Configuration — SR-IOV Logical Port

IBM Power SR-IOV is managed at the hypervisor level via the Hardware Management Console (HMC). The LPAR initially had only a virtual NIC (`env32`, `ibmveth` driver) with no SR-IOV capability.

**HMC Steps performed:**
1. Navigated to: Systems → LTC13U27-Ranier → LPAR `onprem133725233` → Hardware Virtualized I/O
2. Selected **Add SR-IOV Logical Port**
3. Selected **Logical Port Type: Ethernet**
4. Selected Physical Port: `U78DA.ND0.WZS04W2-P0-C7-T0` (Link Status: Up, 20% available capacity)
5. Configuration applied:
   - Logical Port Capacity: 2%
   - Promiscuous Mode: **Disabled** (only 1 promiscuous port allowed per physical port — already used by VIOS)
   - MAC Address Restrictions: Allow all
   - VLAN ID Restrictions: Allow all

**Note:** First attempt failed with `HSCL1257` — promiscuous mode limit exceeded. Resolved by disabling promiscuous mode.

**Confirmed HMC port assignment (Hardware Virtualized I/O → SR-IOV Logical Ports table):**

| Adapter ID | Physical Port ID | Device Name | Location Code | Capacity (%) | Type |
|---|---|---|---|---|---|
| 2 | 0 | `enP32770p1s0` | U78DA.ND0.WZS04W2-P0-C7-T0-S2 | 2 | Ethernet |

**Verification after HMC assignment:**
```bash
ip link show
# 3704: enP32770p1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP

for iface in $(ls /sys/class/net/ | grep -v lo); do
  echo -n "$iface: "
  cat /sys/class/net/$iface/device/uevent 2>/dev/null | grep DRIVER || echo "virtual"
done
# enP32770p1s0: DRIVER=mlx5_core   ← SR-IOV Logical Port confirmed
# env32: DRIVER=ibmveth             ← virtual NIC unchanged
```

**VFIO modules verified:**
```bash
modprobe vfio-pci
lsmod | grep vfio
# vfio_pci, vfio_pci_core, vfio_iommu_spapr_tce, vfio, iommufd — all loaded

dmesg | grep -i iommu
# pci 8002:01:00.0: Adding to iommu group 0
# mlx5_core 8002:01:00.0: iommu: 64-bit OK
```

---

### 3.7 SR-IOV Network Operator Installation

**Repository:** `https://github.com/k8snetworkplumbingwg/sriov-network-operator`  
**Clone:** `git clone https://github.com/k8snetworkplumbingwg/sriov-network-operator` (latest stable release)

**ppc64le image availability verified:**
```bash
podman manifest inspect ghcr.io/k8snetworkplumbingwg/sriov-network-operator:v1.6.0 \
  | python3 -m json.tool | grep -i "architecture"
# "architecture": "ppc64le"  ← confirmed
```

All three core images confirmed ppc64le support:
- `ghcr.io/k8snetworkplumbingwg/sriov-network-operator:v1.6.0`
- `ghcr.io/k8snetworkplumbingwg/sriov-cni:v2.9.0`
- `ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:v3.9.0`

**Installation issues encountered:**

1. **`make deploy-setup-k8s` failed — Go build error on ppc64le:**
   ```
   golang.org/x/tools@v0.16.1/internal/tokeninternal/tokeninternal.go:78:9:
   invalid array length -delta * delta (constant -256 of type int64)
   make: *** [Makefile:148: controller-gen] Error 1
   ```
   The Makefile builds `controller-gen` from source. `golang.org/x/tools v0.16.1` has a ppc64le-incompatible constant expression. This is an upstream Go tooling bug.

2. **`kustomize build config/crd` failed:**  
   `config/crd/` directory exists but contains no `kustomization.yaml`. The pre-built CRD yaml files exist in `config/crd/bases/`.

**Resolution — deploy without make(If "make deploy-setup-k8s" fails) :**
```bash
# Install kustomize pre-built binary for ppc64le
curl -sL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.3.0/kustomize_v5.3.0_linux_ppc64le.tar.gz" \
  | tar -xz -C /usr/local/bin/

# Apply CRDs directly
kubectl apply -f config/crd/bases/

# Set variables and run deploy script directly
export NAMESPACE=sriov-network-operator
export ADMISSION_CONTROLLERS_ENABLED=false
export OPERATOR_EXEC=kubectl
export CLUSTER_TYPE=kubernetes
source hack/env.sh
bash hack/deploy-setup.sh sriov-network-operator
```

**Deployment result:**
```bash
kubectl get -n sriov-network-operator all
# NAME                                          READY   STATUS
# pod/sriov-network-config-daemon-9vsbc         1/1     Running
# pod/sriov-network-operator-6c8c9cdb7d-rc5kw   1/1     Running
#
# daemonset.apps/sriov-network-config-daemon    DESIRED:1 READY:1
# deployment.apps/sriov-network-operator        READY:1/1
```

---

## 4. Observations

| # | Observation | Status |
|---|---|---|
| 1 | CentOS Stream 10 requires manual loading of iptables kernel modules for Calico | Resolved |
| 2 | KubeVirt deployed successfully on ppc64le using `quay.io/pkenchap` images | ✅ Working |
| 3 | Test VM (`testvmfedora`) started successfully, console accessible | ✅ Working |
| 4 | Multus CNI deployed successfully, wrapping Calico as primary CNI | ✅ Working |
| 5 | SR-IOV Logical Port (`enP32770p1s0`) created via HMC and visible in OS | ✅ Working |
| 6 | VFIO modules (`vfio_pci`, `vfio_iommu_spapr_tce`) loaded successfully | ✅ Working |
| 7 | SR-IOV Operator `make deploy-setup-k8s` fails on ppc64le — Go tooling bug | ❌ Blocked |
| 8 | SR-IOV Operator deployed successfully via manual script path | ✅ Working |
| 9 | `sriov-network-config-daemon` started and `SriovNetworkNodeState` CR created | ✅ Working |
| 10 | `syncStatus: Succeeded` — daemon ran without errors | ✅ Working |
| 11 | No SR-IOV interfaces discovered in `SriovNetworkNodeState` | ❌ Blocked |
| 12 | Mellanox VF (15b3:1016) rejected as "unsupported model" | ❌ Blocked |

---

## 5. Initial Investigation — Why SR-IOV Device Is Not Discovered

The `sriov-network-config-daemon` logs show the device `enP32770p1s0` (Mellanox, vendor `15b3`, device ID `1016`) is being detected on the node but rejected during discovery:

```
IsSupportedModel(): found unsupported model  {"vendorId": "15b3", "deviceId": "1016"}
DiscoverSriovDevices(): unsupported device   8002:01:00.0 — MT27710 ConnectX-4 Lx Virtual Function
```

The operator maintains a `supported-nic-ids` ConfigMap which lists known SR-IOV capable NIC models. The device ID `1016` reported by the IBM Power SR-IOV Logical Port does not match the expected entry for this adapter family in the supported list.

The IBM Power SR-IOV Logical Port is a pre-allocated Virtual Function managed at the HMC hypervisor level. It is possible that the operator's NIC discovery model does not account for this IBM Power-specific VF-only configuration, where the Physical Function is not exposed to the OS. Further investigation is needed to confirm whether this is a supported NIC list gap, a configuration issue, or a deeper architectural incompatibility.

---

## 6. Supporting Evidence

### 6.1 Config Daemon Log — Device Rejected
```
IsSupportedModel(): found unsupported model
  {"vendorId:": "15b3", "deviceId:": "1016"}

DiscoverSriovDevices(): unsupported device
  {"device": "8002:01:00.0 -> driver: 'mlx5_core'
   class: 'Network controller'
   vendor: 'Mellanox Technologies'
   product: 'MT27710 Family [ConnectX-4 Lx Virtual Function]'"}
```

### 6.2 SriovNetworkNodeState — Empty Interfaces
```yaml
status:
  syncStatus: Succeeded
  system:
    rdmaMode: shared
  # interfaces: []  ← absent — no SR-IOV devices discovered
```

### 6.3 Device Verification Commands
```bash
# Interface driver confirmation
ethtool -i enP32770p1s0
# driver: mlx5_core
# bus-info: 8002:01:00.0

# Device type confirmation — IS a VF, NOT a PF
cat /sys/class/net/enP32770p1s0/device/physfn 2>/dev/null && echo "IS A VF"
# IS A VF

cat /sys/class/net/enP32770p1s0/device/sriov_numvfs 2>/dev/null || echo "not a PF"
# not a PF

# PCI ID confirmation
cat /sys/class/net/enP32770p1s0/device/uevent | grep PCI_ID
# PCI_ID=15B3:1016    ← VF device ID
```

### 6.4 Supported NIC List — VF ID Not in PF Column
```
Nvidia_mlx5_ConnectX-4LX: 15b3 1015 1016
#                               ^^^^  PF ID (1015) ← what operator checks
#                                     ^^^^  VF ID (1016) ← what IBM Power exposes
```

### 6.5 Cluster and Pod Status
```bash
kubectl get nodes
# NAME                                    STATUS   ROLES                  VERSION
# onprem133725233.pokprv.stglabs.ibm.com  Ready    control-plane,worker   v1.33.x

kubectl get -n sriov-network-operator all
# pod/sriov-network-config-daemon-9vsbc         1/1   Running
# pod/sriov-network-operator-6c8c9cdb7d-rc5kw   1/1   Running
# daemonset/sriov-network-config-daemon         DESIRED:1 READY:1
# deployment/sriov-network-operator             READY:1/1
```

---

## 7. Current Status

### 7.1 What Is Working

| Component | Status | Notes |
|---|---|---|
| kubeadm single-node cluster | ✅ Working | Node Ready, v1.33 |
| Calico CNI | ✅ Working | Requires manual iptables module loading |
| KubeVirt | ✅ Working | Deployed via ppc64le fork (quay.io/pkenchap) |
| KubeVirt test VM | ✅ Working | Fedora 39 VM running, console accessible |
| Multus CNI | ✅ Working | Thick plugin, wrapping Calico |
| SR-IOV Logical Port | ✅ Working | enP32770p1s0, mlx5_core, IOMMU group 0 |
| VFIO modules | ✅ Working | vfio_pci + vfio_iommu_spapr_tce loaded |
| SR-IOV Operator deployment | ✅ Working | Both pods Running via manual script path |

### 7.2 What Is Blocked

| Blocker | Root Cause | Upstream Fix Required |
|---|---|---|
| `make deploy-setup-k8s` fails | `golang.org/x/tools v0.16.1` ppc64le bug in `tokeninternal.go` | Fix in golang.org/x/tools or Makefile to use pre-built binaries |
| SR-IOV device not discovered | `IsSupportedModel()` rejects VF device ID (1016) — expects PF device ID (1015) | Add IBM Power VF-only device support to operator discovery logic |
| No SR-IOV resources in K8s | Cascades from device not discovered | Blocked by above |
| KubeVirt VM SR-IOV passthrough | No SR-IOV resources available for scheduling | Blocked by above |

### 7.3 Summary Assessment

The SR-IOV Network Operator v1.6.0 **deploys and runs successfully** on IBM Power ppc64le. The operator's infrastructure (pods, CRDs, daemon) functions correctly. However, the operator **cannot discover IBM Power SR-IOV Logical Ports** because its NIC detection model assumes x86-style SR-IOV where the Physical Function (PF) is visible to the OS.

On IBM Power, the PF is managed exclusively by the PowerVM hypervisor (HMC). The OS only sees the pre-allocated Virtual Function (Logical Port), which presents with the VF device ID (`1016`) not the PF device ID (`1015`). The operator's `IsSupportedModel()` function does not handle this VF-only configuration.

**Proposed upstream fix:** The `supported-nic-ids` ConfigMap and `IsSupportedModel()` logic would need to support a VF-only mode where the presented device ID matches the VF device ID column (not the PF device ID column). Alternatively, a new configuration path for IBM Power Logical Ports could be introduced in the operator.

---

