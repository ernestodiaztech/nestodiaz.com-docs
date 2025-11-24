---
sidebar_position: 6
---

# 6. NFS Client Deployment

## Prerequisites

This guide is written for a Red Hat Enterprise Linux 8 based operating system which is operating within a cluster of systems.

## Deployment Steps

:::note

Instructions assume execution using the `root` account.

:::

:::warning

When NFS mounting the `/home` directory ensure that only one mount for `/home` is defined in `/etc/fstab`.

:::

>1. Install required NFS client packages.
>
>```json
>dnf -y install nfs-utils
>```

>2. Set the Domain Name:
>
>```json
>sed -i 's/#Domain = local.domain.edu/Domain = nestodiaz.com/g' /etc/idmapd.conf
>```

>3. Create local directories:
>
>```json
>mkdir -p /app
>mkdir -p /home
>mkdir -p /scratch
>```

>4. Add the NFS mounts to `/etc/fstab` to enable them at boot:
>
>```json
>cat >> /etc/fstab <<EOL
>
># NFS Mounts
>nfs01.nestodiaz.com:/srv/nfs/app     /app     nfs4 defaults,tcp,soft,nfsvers=4 0 0
>nfs01.nestodiaz.com:/srv/nfs/home    /home    nfs4 defaults,tcp,soft,nfsvers=4 0 0
>nfs01.nestodiaz.com:/srv/nfs/scratch /scratch nfs4 defaults,tcp,soft,nfsvers=4 0 0
>
>EOL
>```

>5. Reload the fstab in systemd:
>
>```json
>systemctl daemon-reload
>```

>6. Mount the NFS paths:
>
>```json
>mount /app
>mount /home
>mount /scratch
>```