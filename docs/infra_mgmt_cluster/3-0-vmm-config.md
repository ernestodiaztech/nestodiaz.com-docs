---
sidebar_position: 3
---

# 3. Virtual Machine Manager Configuration

The following instructions are for configuring the Virtual Machine Managers.

## Prerequisites

This guide is written for a Red Hat Enterprise Linux 8 based operating system which is operating within a cluster of systems and the following are the prerequisites:

- [Virtual Machine Manager OS Installation](/docs/infra_mgmt_cluster/2-0-vmm-install)

## Post Deployment

> The Cockpit Management Web Interface can be accessed at the following URLs:
> Virtual Machine Manager 1: https://vmm01.nestodiaz.com:9090
> Virtual Machine Manager 2: https://vmm02.nestodiaz.com:9090

## Deployment Steps

:::note

Instructions assume execution using the `root` account.

:::

First, I install the dependencies.

```bash
dnf -y install tar openssl-devel cockpit cockpit-packagekit \
    cockpit-pcp cockpit-storaged cockpit-system cockpit-ws \
    cockpit-machines qemu-kvm qemu-kvm-block-iscsi \
    qemu-kvm-block-curl qemu-kvm-common qemu-kvm-block-ssh \
    qemu-kvm-block-iscsi lm_sensors lm_sensors-devel lm_sensors-libs \
    virt-install libosinfo
```

Then I enabled Cockpit.

```bash
systemctl enable --now cockpit.socket
firewall-cmd --zone=public --add-service=cockpit --permanent
systemctl reload firewalld
```

I then created `guest_images` storage pool.

```bash
# Create KVM directories
mkdir -p /srv/kvm
mkdir    /srv/kvm/iso
mkdir    /srv/kvm/img
mkdir    /srv/tmp

# Create pools
virsh pool-define-as "guest_images" dir - - - - "/srv/kvm/img/"
virsh pool-build 'guest_images'
virsh pool-start 'guest_images'
virsh pool-autostart 'guest_images'
```

I then created storage volumes.

```bash title="Virtual Machine Manager 1"
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/idm.qcow2     96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/gitlab.qcow2  2T
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/slurm.qcow2   96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/graylog.qcow2 96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/ansible.qcow2 96G
```

```bash title="Virtual Machine Manager 2"
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/indfuxdb.qcow2 96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/grafana.qcow2  96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/docker.qcow2   2T
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/mirror.qcow2   96G
qemu-img create -f qcow2 -o preallocation=off /srv/kvm/img/vmg01.qcow2    96G
```

I then created a virtual machine bridge.

:::important

Verify the name of the ethernet interface and the IPv4 address. The interface name may be different than “eno1” and the IPv4 addresses should be the value for your environment.

:::

For Virtual Machine Manager 1:

```bash
IP_ADDRESS="10.33.99.151"
```

For Virtual Machine Manager 2:

```bash
IP_ADDRESS="10.33.99.152"
```

For both Virtual Machine Manager 1 and Virtual Machine Manager 2:

```bash
# Bridge Configuration
ETHERNET_INTERFACE="eno1"
BRIDGE_NAME="vmbr0"
IP_DNS="192.168.1.1"
IP_GATEWAY="192.168.1.1"

# (Informational Only) List Interfaces
ip addr

# (Informational Only) List Active Network Connections
nmcli conn show

# Create a bridge interface
nmcli connection add type bridge con-name ${BRIDGE_NAME} ifname ${BRIDGE_NAME}

# Add static IP address
nmcli conn modify ${BRIDGE_NAME} ipv4.addresses "${IP_ADDRESS}/24"
nmcli conn modify ${BRIDGE_NAME} ipv4.gateway "${IP_GATEWAY}"
nmcli conn modify ${BRIDGE_NAME} ipv4.dns "${IP_DNS}"
nmcli conn modify ${BRIDGE_NAME} ipv4.method manual

# Assign the interfaces to the bridge
nmcli connection add type ethernet slave-type bridge autoconnect yes \
    con-name bridge-${BRIDGE_NAME} ifname ${ETHERNET_INTERFACE} master ${BRIDGE_NAME}

# Bring up or activate the bridge connection
nmcli conn up ${BRIDGE_NAME}

# Bring down wired connection
nmcli conn down ${ETHERNET_INTERFACE}

# (Informational Only) Display the network interfaces
nmcli device status

# (Informational Only) List Interfaces
ip addr

# (Informational Only) Show Bridge Details
nmcli -f bridge con show ${BRIDGE_NAME}

# Declaring the KVM Bridged Network
virsh net-list --all

cat << 'EOL' > /tmp/bridge.xml
<network>
  <name>vmbr0</name>
  <forward mode="bridge"/>
  <bridge name="vmbr0"/>
</network>
EOL

virsh net-define /tmp/bridge.xml
virsh net-start ${BRIDGE_NAME}
virsh net-autostart ${BRIDGE_NAME}
```

I then uploaded the guest ISO images.

:::important

The ISO file name will vary depending OS and version being used.

:::

```bash
scp rhel-8.8-x86_64-dvd.iso root@vmm1.nestodiaz.com:/srv/kvm/iso/rhel-8.8-x86_64-dvd.iso
scp rhel-8.8-x86_64-dvd.iso root@vmm2.nestodidaz.com:/srv/kvm/iso/rhel-8.8-x86_64-dvd.iso
```

I then suppressed negotiate headers.

:::note

This is optional and prevents falling back to HTML login boxes in Windows browsers.

:::

```bash
sudo nano /etc/cockpit/cockpit.conf
```

```bash
[gssapi]
action = none

[negotiate]
action = none
```

I then rebooted the system.

```bash
sudo reboot now
```

After the reboot, I logged into Cockpit.

