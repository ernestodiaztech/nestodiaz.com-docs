---
sidebar_position: 5
---

# 5. NFS Server Deployment

The following instructions are for deploying the NFS server.

## Prerequisites

This guide is written for a Red Hat Enterprise Linux 8 based operating system which is operating within a cluster of systems.

## Post Deployment

The Cockpit Management Web Interface can be accessed at the following URLs: NFS Server Node https://nfs01.nestodiaz.com:9090

## Deployment Steps

:::note

Instruction assume execution using the `root` account.

:::

1. Install required NFS server packages:

```json
dnf -y install nfs-utils
```

2. Enable Cockpit

```json
dnf -y remove cockpit-podman
systemctl enable --now cockpit.socket
```

3. Set the Domain Name:

```json
sed -i 's/#Domain = local.domain.edu/Domain = nestodiaz.com/g' /etc/idmapd.conf
```

4. Create directories that will be exported by NFS:

```json
# Create Directories
mkdir -p /srv/nfs/app
mkdir -p /srv/nfs/backup
mkdir -p /srv/nfs/home
mkdir -p /srv/nfs/mirror
mkdir -p /srv/nfs/scratch

# Set owner to root
chown -R root:root /srv/nfs/app
chown -R root:root /srv/nfs/backup
chown -R root:root /srv/nfs/home
chown -R root:root /srv/nfs/mirror
chown -R root:root /srv/nfs/scratch

# Limit file and directory creation to root at this level
chmod 755 /srv/nfs/app
chmod 755 /srv/nfs/backup
chmod 755 /srv/nfs/home
chmod 755 /srv/nfs/mirror
chmod 755 /srv/nfs/scratch

# Set SELinux context
chcon -R -t user_home_dir_t /srv/nfs/home
semanage fcontext -a -t user_home_dir_t /srv/nfs/home
```

5. Create exports file:

```json
sh -c 'cat >> /etc/exports <<EOL

# NFS Exports
/srv/nfs/app     192.168.1.0/24(rw,sync,secure,wdelay,no_subtree_check,no_root_squash)
/srv/nfs/backup  192.168.1.0/24(rw,sync,secure,wdelay,no_subtree_check,no_root_squash)
/srv/nfs/home    192.168.1.0/24(rw,sync,secure,wdelay,no_subtree_check,no_root_squash)
/srv/nfs/mirror  192.168.1.0/24(rw,sync,secure,wdelay,no_subtree_check,no_root_squash)
/srv/nfs/scratch 192.168.1.0/24(rw,sync,secure,wdelay,no_subtree_check,no_root_squash)

EOL'
```

6. Start the NFS services:

```json
systemctl enable --now rpcbind nfs-server
```

7. Configure firewall rules:

:::note

The additional firewalld rule `firewall-cmd --zone=nfs-server --add-service={nfs3,mountd,rpc-bind} --permanent` is required if supporting NFSv3.

:::

```json
systemctl enable --now firewalld
firewall-cmd --zone=public --add-source=192.168.1.0/24 --permanent
firewall-cmd --zone=public --add-service=cockpit --permanent
firewall-cmd --zone=public --add-service=nfs --permanent
firewall-cmd --reload
```