# Migration Plan: NFS + DRBD + Keepalived to Pacemaker (Zero Downtime)

## Environment Details

* **Primary Node (current active)**: `dea2coknfs-0101`
* **Secondary Node (initial migration target)**: `dea2coknfs-0102`
* **Virtual IP (VIP)**: `10.76.172.7`
* **DRBD Resource**: `boomi`
* **DRBD Device**: `/dev/drbd0`
* **LV Path**: `/dev/drbd-boomi/lv-boomi`
* **Mount Point**: `/mnt/boomi`
* **Filesystem**: `xfs`
* **Kubernetes NFS Clients**: Should mount via `10.76.172.7:/mnt/boomi`

---

## STEP 1: Pre-Migration Checks (On Both Nodes)

```bash
cat /proc/drbd
```

Ensure output includes:

```
cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate
```

Ensure both nodes are fully synchronized.

---

## STEP 2: Prepare Worker Node (dea2coknfs-0102)

### 2.1 Install Required Packages

```bash
apt update
apt install -y pacemaker corosync fence-agents fence-agents-vmware-common
```

### 2.2 Stop Keepalived & NFS Services

```bash
systemctl stop keepalived nfs-server
systemctl disable keepalived nfs-server
```

### 2.3 Configure `pcs` User Password (on both nodes)

run passwd hacluster in both nodes and set password manually 

```bash
echo u hacluster:StrongPassw0rd | chpasswd
```

---

## STEP 3: Configure Pacemaker Cluster (From dea2coknfs-0102)

```bash
pcs cluster auth dea2coknfs-0101 dea2coknfs-0102 -u hacluster -p StrongPassw0rd
pcs cluster setup --name nfs-ha-cluster dea2coknfs-0101 dea2coknfs-0102
pcs cluster start --all
pcs cluster enable --all
```

---

## STEP 4: Setup Fencing (STONITH via vSphere)

```bash
pcs stonith create vm-fence fence_vmware_soap \
  ipaddr=<vcenter-ip-or-hostname> \
  login=<vcenter-user> \
  passwd=<vcenter-pass> \
  ssl_insecure=1 \
  pcmk_host_map="dea2coknfs-0101:vm01;dea2coknfs-0102:vm02" \
  pcmk_host_check=static-list \
  op monitor interval=60s
```

You can provide either the **vCenter IP address or hostname** for the `ipaddr` parameter. This information is required because Pacemaker uses the vCenter API to power off or reset the VM during a fencing operation, ensuring high availability and preventing split-brain.

Replace `<vcenter-ip-or-hostname>`, `<vcenter-user>`, `<vcenter-pass>`, and VM names accordingly.

---

## STEP 5: Define Cluster Resources (Run on dea2coknfs-0102)

```bash
# Run these commands on dea2coknfs-0102 only

# DRBD resource (Master/Slave)
pcs resource create drbd-boomi ocf:linbit:drbd drbd_resource=boomi \
  op monitor interval=30s role=Master \
  op monitor interval=60s role=Slave \
  promotable

# Filesystem (use full agent name; package may be missing)
# If this fails, ensure the 'resource-agents' package is installed:
# apt install resource-agents

pcs resource create fs-boomi ocf:heartbeat:Filesystem \
  device="/dev/drbd0" directory="/mnt/boomi" fstype="xfs"

# NFS Server
pcs resource create nfs-server systemd:nfs-server op monitor interval=30s

# Virtual IP
pcs resource create vip-boomi ocf:heartbeat:IPaddr2 \
  ip=10.76.172.7 cidr_netmask=24 interface=ens192 op monitor interval=15s

# NFS Export (optional)
# Note: 'exportfs' OCF agent is not available by default in all distros.
# If you encounter metadata or agent not found errors, you can skip this
# and instead manage NFS exports using /etc/exports + exportfs -a

# If available:
# pcs resource create nfs-export-boomi ocf:heartbeat:exportfs \
#   clientspec="*" \
#   directory="/mnt/boomi" \
#   options="rw,no_root_squash,no_subtree_check" \
#   fsid=0
```

```
```

---

## STEP 6: Set Resource Ordering & Colocation

```bash
# Use correct clone ID and updated role syntax

pcs constraint colocation add fs-boomi with drbd-boomi-clone INFINITY
pcs constraint colocation add nfs-server with fs-boomi INFINITY
pcs constraint colocation add vip-boomi with nfs-server INFINITY
# Only add exportfs constraint if exportfs is available
# pcs constraint colocation add nfs-export-boomi with nfs-server INFINITY

pcs constraint order promote drbd-boomi-clone then start fs-boomi
pcs constraint order start fs-boomi then start nfs-server
pcs constraint order start nfs-server then start vip-boomi
# pcs constraint order start vip-boomi then start nfs-export-boomi
```

Note:

* Use `drbd-boomi-clone` (not `drbd-boomi`) in constraints because Pacemaker treats promotable DRBD as a clone.
* Replace deprecated role `master` with `Promoted` in all future constraints (Pacemaker supports both but warns on old syntax).
* Skip the `nfs-export-boomi` constraints unless the exportfs agent is correctly configured.bash
  pcs constraint colocation add fs-boomi with master drbd-boomi INFINITY
  pcs constraint colocation add nfs-server with fs-boomi INFINITY
  pcs constraint colocation add vip-boomi with nfs-server INFINITY
  pcs constraint colocation add nfs-export-boomi with nfs-server INFINITY

pcs constraint order promote drbd-boomi then start fs-boomi
pcs constraint order start fs-boomi then start nfs-server
pcs constraint order start nfs-server then start vip-boomi
pcs constraint order start vip-boomi then start nfs-export-boomi

````

---

## STEP 7: Controlled Failover from dea2coknfs-0101 to Pacemaker (On dea2coknfs-0101)

```bash
# Step into maintenance
systemctl stop keepalived nfs-server
umount /mnt/boomi
drbdadm secondary boomi
````

```bash
# On dea2coknfs-0102
pcs resource enable drbd-boomi
```

Verify:

```bash
pcs status
showmount -e 10.76.172.7
mount | grep boomi
```

---

## STEP 8: Finish Migration (Master Node dea2coknfs-0101)

### 8.1 Install Cluster Stack

```bash
apt install -y pacemaker corosync fence-agents fence-agents-vmware-common
```

### 8.2 Ensure services are stopped

```bash
systemctl stop keepalived nfs-server
systemctl disable keepalived nfs-server
```

### 8.3 Verify Node Joins Cluster

```bash
pcs status nodes
pcs cluster start dea2coknfs-0101
```

---

## STEP 9: Test Failover

```bash
pcs node standby dea2coknfs-0102  # Force failover to dea2coknfs-0101
pcs node unstandby dea2coknfs-0102
```

Check:

```bash
pcs status
cat /proc/drbd
showmount -e 10.76.172.7
```

---

## ✅ Final Checklist

*

---

Migration complete.
