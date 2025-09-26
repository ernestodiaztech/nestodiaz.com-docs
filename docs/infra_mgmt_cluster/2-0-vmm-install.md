---
sidebar_position: 2
---

# 2. Virtual Machine Manager OS Installation

The following instructions are for installing the operating system on the bare-metal Virtual Machine Manager nodes.

## Configuration

<table>
  <thead>
    <tr>
      <th colspan="2">Virtual Machine Manager 1</th>
      <th colspan="2">Virtual Machine Manager 2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>OS</strong></td>
      <td>Red Hat Enterprise Linux 8</td>
      <td><strong>OS</strong></td>
      <td>Red Hat Enterprise Linux 8</td>
    </tr>
    <tr>
      <td><strong>Installation Type</strong></td>
      <td>Bare-Metal</td>
      <td><strong>Installation Type</strong></td>
      <td>Bare-Metal</td>
    </tr>
    <tr>
      <td><strong>Hostname</strong></td>
      <td>vmm01</td>
      <td><strong>Hostname</strong></td>
      <td>vmm02</td>
    </tr>
    <tr>
      <td><strong>FQDN</strong></td>
      <td>vmm01.nestodiaz.com</td>
      <td><strong>FQDN</strong></td>
      <td>vmm02.nestodiaz.com</td>
    </tr>
  </tbody>
</table>

## OS Installation Steps

1. Boot system using Red Hat Enterprise Linux (RHEL) 8.8 ISO
2. Select `Install Red Hat Enterprise Linux 8.8` and press `ENTER`
3. Select your language to use during the installation process and press `Continue`

:::note

You should not see the `Installation Summary` GUI.

:::

4. Click on Language Support, set as needed:

```bash
* Select your language to be used following installation
* Click on the "Done" button to return
```

5. Click on `Network & Host Name`, set the following:

:::note

The name of the ethernet interface (eno1) may be different.

:::

```bash
* For Virtual Machine Manager Node 1, use Host Name: vmm01
* For Virtual Machine Manager Node 2, use Host Name: vmm02
* Click on the "Configure..." button

    1. Under the General tab, Check "Connect automatically with priority"
    2. Under the Ethernet tab, Set Link negotiation to "Automatic"
    3. Under the IPv4 Settings tab,

        * Select Method: "Manual"
        * Set Address, Netmask, and Gateway
        * Set DNS servers

    4. Under IPv6 Settings tab,

        * Select Method "Link-Local Only"

    5. Click on the "Save" button

* Turn Ethernet (eno1) "On"
* Hit the "Apply" button to set the host name
* Click on the "Done" button to return
```

6. Click on `Time & Date`, set the following:

```bash
* Select your timezone
* Enable "Network Time"
* Configure NTP servers as needed
* Click on the "Done" button to return
```

7. Click on `Installation Source`, set the following:

```bash
* Click on "Auto-detected installation media"
* Click on the "Done" button to return
```

8. Click on `Software Selection`, set the following:

```bash
* Select Base Environment: "Virtualization Host"

    1. Check: "Network File System Client"
    2. Check: "Remote Manager for Linux"
    3. Check: "Virtualization Platform"
    4. Check: "Security Tools"
    5. Check: "System Tools"

* Click on the "Done" button to return
```

9. Click on Installation Destination, set the following:

```bash
* Use "Custom" Storage Configuration
* Click on the "Done" button to continue
* Configure using below Partition Table
* Click on the "Done" button to continue
* Click on the "Accept Changes" button to continue
```

>### Partition Table
>
>:::note
>
>This partition table is based on the requirements of the NIST security profile. It is the recommended layout even if the profile is not enabled to ensure future compatibility with the security profiles. Adjust the sizes of the partions as needed.
>
>:::
>
>:::note
>
>Path `/srv` will be used to store the hosted application VM images and the size will need to be adjusted to reflect your hardware configuration. The `/srv` path should be configured on local storage that utilizes a RAID-6 volume.
>
>:::
>
>| Partition Name   | Size  | Device Type        | Drives       | File System            |
>|------------------|-------|-------------------|--------------|------------------------|
>| /boot/efi        | 2 GiB | Standard Partition | OS (RAID-1)  | EFI System Partition   |
>| /boot            | 2 GiB | Standard Partition | OS (RAID-1)  | xfs                    |
>| /                | 32 GiB| LVM                | OS (RAID-1)  | xfs                    |
>| /home            | 2 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /srv             | 24 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /tmp             | 8 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var             | 32 GiB| LVM                | OS (RAID-1)  | xfs                    |
>| /var/log         | 5 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var/log/audit   | 5 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var/tmp         | 8 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| swap             | 2 GiB | LVM                | OS (RAID-1)  | swap                   |

10. Click on KDUMP, set the following:

```bash
* Uncheck "Enable kdump"
* Click on the "Done" button to return
```

11. Click on `Root Password`, set the following:

>:::note
>
>Only a root account will be created.
>
>:::
>
>```bash
>* Set a strong Root password
>* Click on the "Done" button to return
>```

12. Click on the Begin Installation button to begin
13. Click on the Reboot System button when installation has completed