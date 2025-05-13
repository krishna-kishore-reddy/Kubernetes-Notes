# Complete DRBD + NFS + Keepalived High Availability Setup Guide

This guide provides detailed, step-by-step instructions for setting up a highly available NFS server using DRBD for data replication and Keepalived for service failover. This setup creates a RAID-1 like system across two nodes, ensuring both data redundancy and service availability.

## Environment Setup

- **Two Ubuntu servers:**
  - Node1 (Primary): 192.168.1.1
  - Node2 (Backup): 192.168.1.2
- **Floating IP:** 192.168.1.10
- **Storage:** Each server has an additional disk or partition (we'll use /dev/sdb)
- **OS:** Ubuntu 22.04 LTS

## Step 1: Initial Setup on Both Nodes

First, update and install required packages on both servers:

```bash
# Update package lists
sudo apt update

# Install required packages
echo "postfix postfix/main_mailer_type select No configuration" | sudo debconf-set-selections
sudo apt install -y nfs-kernel-server drbd-utils keepalived

# Make sure services are not running yet
sudo systemctl stop nfs-kernel-server
sudo systemctl stop drbd
```

## Step 2: Configure Hosts File on Both Nodes

Update the hosts file to ensure both nodes can resolve each other by name:

```bash
sudo nano /etc/hosts
```

Add:
```
10.72.172.25 usr2coknfs02
10.72.172.26 usr2coknfs03
```

## Step 3: Prepare the Disk for DRBD

Ensure you have an available disk or partition. If using a whole disk, partition it:

```bash
sudo fdisk /dev/sdb
```

Create a new primary partition by:
1. Press `n` for new partition
2. Press `p` for primary
3. Accept defaults for partition number and size
4. Press `w` to write changes

## Step 4: Configure DRBD

Create the DRBD configuration file on both nodes:

```bash
sudo nano /etc/drbd.d/nfs-data.res
```

Add:
```
resource nfs-data {
  device /dev/drbd0;
  disk /dev/sdb1;  # Change if your partition is different
  meta-disk internal;
  
  net {
    protocol C;
    max-buffers 8000;
    max-epoch-size 8000;
  }
  
  on usr2coknfs02 {
    address 10.72.172.25:7789;
  }
  
  on usr2coknfs03 {
    address 10.72.172.26:7789;
  }
  
  syncer {
    rate 100M;
    verify-alg sha1;
  }
  
  disk {
    on-io-error detach;
  }
}
```

## Step 5: Initialize DRBD Resource

Execute these commands on both nodes:

```bash
# Create DRBD metadata
sudo drbdadm create-md nfs-data

# If you get a warning about existing data, force it:
# sudo drbdadm create-md --force nfs-data

# Start DRBD
sudo drbdadm up nfs-data
```

## Step 6: Set Up Initial Synchronization

Only on Node1 (primary):

```bash
# Make Node1 primary
sudo drbdadm primary --force nfs-data

# Create a filesystem on the DRBD device
sudo mkfs.ext4 -F /dev/drbd0

# Create mount point for NFS share
sudo mkdir -p /data/nfs_share

# Mount the DRBD device
sudo mount /dev/drbd0 /data/nfs_share

# Set permissions
sudo chown nobody:nogroup /data/nfs_share
sudo chmod 777 /data/nfs_share
```

Only on Node2 (backup):

```bash
# Create the mount point directory (do not mount it yet)
sudo mkdir -p /data/nfs_share
```

Monitor synchronization status on both nodes:

```bash
sudo cat /proc/drbd
```

Wait until both nodes show `UpToDate/UpToDate` before proceeding.

## Step 7: Configure NFS Server

On both nodes:

```bash
sudo nano /etc/exports
```

Add:
```
/data/nfs_share *(rw,sync,no_subtree_check,no_root_squash,fsid=0)
```

Configure NFS for better availability:

```bash
sudo nano /etc/default/nfs-kernel-server
```

```bash
sudo vim /etc/nfs.conf
[mountd]
port=892

```

Add or modify:
```
RPCMOUNTDOPTS="--manage-gids -p 892"
RPCNFSDOPTS="-N 2 -N 3 -p 2049"
NEED_STATD=yes
NEED_IDMAPD=yes
```

On the primary node (Node1) only:

```bash
# Export the shares
sudo exportfs -a

# Start NFS server
sudo systemctl start nfs-kernel-server
```

## Step 8: Create Failover Scripts

### Primary Promotion Script

Create this script on both nodes:

```bash
sudo nano /usr/local/bin/drbd-primary.sh
```

Add:
```bash
#!/bin/bash
set -e

LOG="/var/log/drbd-failover.log"
echo "$(date): STARTING PROMOTION TO PRIMARY" >> $LOG

# Make sure we're in a clean state
echo "$(date): Stopping NFS..." >> $LOG
systemctl stop nfs-kernel-server

# Unmount if it's somehow mounted
if mount | grep -q "/data/nfs_share"; then
  echo "$(date): Unmounting existing mount..." >> $LOG
  umount /data/nfs_share
fi

# Check if we're already primary
ROLE=$(drbdadm role nfs-data)
if [[ "$ROLE" != "Primary" ]]; then
  echo "$(date): Making DRBD primary..." >> $LOG
  # Become primary for DRBD resource
  drbdadm primary nfs-data

  # Wait for DRBD to be primary
  COUNT=0
  while ! drbdadm role nfs-data | grep -q "Primary"; do
    sleep 1
    COUNT=$((COUNT+1))
    if [ $COUNT -ge 30 ]; then
      echo "$(date): TIMEOUT waiting for DRBD to become primary" >> $LOG
      exit 1
    fi
  done
else
  echo "$(date): Already in Primary role" >> $LOG
fi

# Check DRBD state
STATE=$(drbdadm dstate nfs-data)
echo "$(date): DRBD state: $STATE" >> $LOG
if [[ "$STATE" != *"UpToDate/UpToDate"* ]]; then
  echo "$(date): WARNING - DRBD not fully synced: $STATE" >> $LOG
fi

# Mount the filesystem
echo "$(date): Mounting DRBD device..." >> $LOG
mount /dev/drbd0 /data/nfs_share
if [ $? -ne 0 ]; then
  echo "$(date): FAILED to mount DRBD device - trying fsck" >> $LOG
  e2fsck -y /dev/drbd0
  mount /dev/drbd0 /data/nfs_share
  if [ $? -ne 0 ]; then
    echo "$(date): FAILED to mount DRBD device even after fsck" >> $LOG
    exit 1
  fi
fi

# Export NFS shares
echo "$(date): Exporting NFS shares..." >> $LOG
exportfs -a

# Start NFS server
echo "$(date): Starting NFS server..." >> $LOG
systemctl start nfs-kernel-server
if [ $? -ne 0 ]; then
  echo "$(date): FAILED to start NFS server" >> $LOG
  systemctl status nfs-kernel-server >> $LOG
  exit 1
fi

echo "$(date): SUCCESSFULLY PROMOTED TO PRIMARY" >> $LOG
```

### Secondary Demotion Script

Create this script on both nodes:

```bash
sudo nano /usr/local/bin/drbd-secondary.sh
```

Add:
```bash
#!/bin/bash

LOG="/var/log/drbd-failover.log"
echo "$(date): STARTING DEMOTION TO SECONDARY" >> $LOG

# Unexport NFS shares
echo "$(date): Unexporting NFS shares..." >> $LOG
exportfs -ua

# Stop NFS server
echo "$(date): Stopping NFS server..." >> $LOG
systemctl stop nfs-kernel-server

# Unmount the filesystem
if mount | grep -q "/data/nfs_share"; then
  echo "$(date): Unmounting DRBD device..." >> $LOG
  umount /data/nfs_share
  if [ $? -ne 0 ]; then
    echo "$(date): WARNING - Failed to unmount cleanly, forcing..." >> $LOG
    umount -f /data/nfs_share
  fi
fi

# Check if we're already secondary
ROLE=$(drbdadm role nfs-data)
if [[ "$ROLE" != "Secondary" ]]; then
  echo "$(date): Making DRBD secondary..." >> $LOG
  # Become secondary for DRBD resource
  drbdadm secondary nfs-data
  
  # Verify we're secondary
  COUNT=0
  while ! drbdadm role nfs-data | grep -q "Secondary"; do
    sleep 1
    COUNT=$((COUNT+1))
    if [ $COUNT -ge 30 ]; then
      echo "$(date): TIMEOUT waiting for DRBD to become secondary" >> $LOG
      exit 1
    fi
  done
else
  echo "$(date): Already in Secondary role" >> $LOG
fi

echo "$(date): SUCCESSFULLY DEMOTED TO SECONDARY" >> $LOG
```

Make both scripts executable:

```bash
sudo chmod +x /usr/local/bin/drbd-primary.sh
sudo chmod +x /usr/local/bin/drbd-secondary.sh
```

## Step 9: Create Monitoring Script

Create this script on both nodes:

```bash
sudo nano /usr/local/bin/monitor-ha.sh
```

Add:
```bash
#!/bin/bash

echo "==== HA Status Report $(date) ===="
echo ""
echo "== DRBD Status =="
cat /proc/drbd
echo ""
echo "== DRBD Role =="
drbdadm role nfs-data
echo ""
echo "== Mount Info =="
mount | grep drbd
echo ""
echo "== NFS Service Status =="
systemctl status nfs-kernel-server | grep Active:
echo ""
echo "== Keepalived Status =="
systemctl status keepalived | grep Active:
echo ""
echo "== NFS Exports =="
exportfs
echo ""
echo "== IP Address (VIP) =="
ip addr | grep 192.168.1.10
echo ""
echo "== Last 10 lines of failover log =="
tail -10 /var/log/drbd-failover.log 2>/dev/null || echo "No log file yet"
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/monitor-ha.sh
sudo touch /var/log/drbd-failover.log
sudo chmod 666 /var/log/drbd-failover.log
```

## Step 10: Configure Keepalived

Configure Keepalived on Node1 (primary):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

Add:
```
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_services {
    script "/bin/true"  # Placeholder script to satisfy keepalived; replace with an actual check if needed
    interval 5
    timeout 5
    weight 2
}

vrrp_instance NFS_HA {
    state MASTER
    interface ens192  # Make sure this is the correct network interface (check with `ip a`)
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass drbd_nfs_secret
    }

    virtual_ipaddress {
        10.72.172.30/24
    }

    track_script {
        check_services
    }

    notify_master "/usr/local/bin/drbd-primary.sh"
    notify_backup "/usr/local/bin/drbd-secondary.sh"
    notify_fault "/usr/local/bin/drbd-secondary.sh"
    notify_stop  "/usr/local/bin/drbd-secondary.sh"
}
```

Configure Keepalived on Node2 (backup):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

Add (almost identical but with BACKUP state and lower priority):
```
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_services {
    script "/bin/true"  # Replace with actual health check script later if needed
    interval 5
    timeout 5
    weight 2
}

vrrp_instance NFS_HA {
    state BACKUP
    interface ens192  # Ensure this matches your actual network interface
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass drbd_nfs_secret
    }

    virtual_ipaddress {
        10.72.172.30/24
    }

    track_script {
        check_services
    }

    notify_master "/usr/local/bin/drbd-primary.sh"
    notify_backup "/usr/local/bin/drbd-secondary.sh"
    notify_fault "/usr/local/bin/drbd-secondary.sh"
    notify_stop  "/usr/local/bin/drbd-secondary.sh"  #added this line after testing, if anything is not working try to delete this line first and check
}
```

## Step 11: Configure Firewall 

If UFW is active, allow necessary traffic:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 2049 proto tcp  # NFS
sudo ufw allow from 192.168.1.0/24 to any port 111 proto tcp   # Portmapper
sudo ufw allow from 192.168.1.0/24 to any port 111 proto udp
sudo ufw allow from 192.168.1.0/24 to any port 892 proto tcp   # Mountd
sudo ufw allow from 192.168.1.0/24 to any port 892 proto udp
sudo ufw allow from 192.168.1.0/24 to any port 7789 proto tcp  # DRBD
sudo ufw allow from 192.168.1.0/24 to any proto 112            # VRRP
```

## Step 12: Start Services

Start Keepalived on both nodes:

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
```

## Step 13: Test the Setup

### Check Initial Status

On both nodes:

```bash
sudo ./usr/local/bin/monitor-ha.sh
```

### Mount NFS on a Client Machine

```bash
# Create a mount point
sudo mkdir -p /mnt/nfs_test

# Mount the NFS share using the floating IP
sudo mount -t nfs -o soft,timeo=10,retrans=2 10.72.172.30:/data/nfs_share /mnt/nfs_test

# Create a test file
touch /mnt/nfs_test/testfile
echo "Hello from NFS" > /mnt/nfs_test/testfile
```

### Test Failover

1. On the primary node, simulate a failure:
   ```bash
   sudo systemctl stop keepalived
   ```

2. Check that the floating IP moved to the secondary node:
   ```bash
   # On secondary node
   ip addr show
   ```

3. Verify NFS service is still accessible on the client:
   ```bash
   cat /mnt/nfs_test/testfile
   # Should show: Hello from NFS
   ```

4. If the client access hangs, retry mounting:
   ```bash
   sudo umount /mnt/nfs_test
   sudo mount -t nfs -o soft,timeo=10,retrans=2 10.72.172.30:/data/nfs_share /mnt/nfs_test
   ```

## Step 14: Configure Client for Persistence

For better failover handling on the client, update `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add:
```
10.72.172.30:/data/nfs_share /mnt/nfs_test nfs rw,soft,timeo=10,retrans=2 0 0
```

## Troubleshooting

### If DRBD Won't Mount After Failover

Try these commands on the node that should be primary:

```bash
# Verify DRBD state
sudo cat /proc/drbd

# If necessary, force primary again
sudo drbdadm primary nfs-data

# Check filesystem
sudo e2fsck -f /dev/drbd0

# Try mounting
sudo mount /dev/drbd0 /data/nfs_share
```

### If NFS Access Fails After Failover

```bash
# On the new primary node:
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# On client:
sudo umount /mnt/nfs_test
sudo mount -t nfs -o soft,timeo=10,retrans=2 10.72.172.30:/data/nfs_share /mnt/nfs_test
```

### Manual Failover Test

To manually test failover:

1. On the currently active node:
   ```bash
   sudo bash /usr/local/bin/drbd-secondary.sh
   ```

2. On the node that should be the new primary:
   ```bash
   sudo bash /usr/local/bin/drbd-primary.sh
   ```

### Check Logs for Issues

```bash
sudo cat /var/log/drbd-failover.log
sudo tail -f /var/log/syslog
```

## Maintenance Notes

- **Always check DRBD status** before performing maintenance: `sudo cat /proc/drbd`
- **Verify which node is primary** before making changes: `sudo drbdadm role nfs-data`
- **Test failover regularly** to ensure the system works as expected
- **Monitor disk space** on the shared filesystem: `df -h /data/nfs_share`
