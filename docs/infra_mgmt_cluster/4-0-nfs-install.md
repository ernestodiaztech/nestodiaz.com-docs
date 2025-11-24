---
sidebar_position: 4
---

# 4. NFS Server OS Installation

The following instructions are for installing the operating system on the bare-metal NFS server node.

## Configuration

<table>
  <thead>
    <tr>
      <th colspan="2">NFS Server</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>OS</strong></td>
      <td>Red Hat Enterprise Linux 8</td>
    </tr>
    <tr>
      <td><strong>Installation Type</strong></td>
      <td>Bare-Metal</td>
    </tr>
    <tr>
      <td><strong>Hostname</strong></td>
      <td>nfs01</td>
    </tr>
    <tr>
      <td><strong>FQDN</strong></td>
      <td>nfs01.nestodiaz.com</td>

    </tr>
  </tbody>
</table>

## OS Installation Steps

>1. Boot system using Red Hat Enterprise Linux (RHEL) 8.8 ISO

>2. Select `Install Red Hat Enterprise Linux 8.8` and press `ENTER`

>3. Select your language to use during the installation process and press `Continue`

>:::note
>
>You should now see the `Installation Summary` GUI.
>
>:::
>
>4. Click on `Language Support`, set the following:
>
>```json
>* Select your language to be used following installation
>* Click on the "Done" button to return
>```

>5. Click on `Network & Host Name`, set the following:
>
>```json
>* Host Name to: nfs01
>* Click on the "Configure..." button
>
>    1. Under the General tab, Check "Connect automatically with priority"
>    2. Under the Ethernet tab, Set Link negotiation to "Automatic"
>    3. Under the IPv4 Settings tab,
>
>        * Select Method: "Manual"
>        * Set IP Address and DNS Servers
>
>    4. Under IPv6 Settings tab,
>
>        * Select Method "Link-Local Only"
>
>    5. Click on the "Save" button
>
>* Turn Ethernet (enp1s0) "On"
>* Hit the "Apply" button to set the host name
>* Click on the "Done" button to return
>```

>6. Click on `Time & Date`, set the following:
>
>```json
>* Select your timezone
>* Enable "Network Time"
>* Configure NTP servers as needed
>* Click on the "Done" button to return
>```

>7. Click on `Installation Source`, set the following:
>
>```jason
>* Click on "Auto-detected installation media"
>* Click on the "Done" button to return
>```

>8. Click on `Software Selection`, set the following:
>
>```json
>* Select Base Environment: "Server"
>
>    1. Check: "File and Storage Server"
>    2. Check: "Network File System Client"
>    3. Check: "Security Tools"
>    4. Check: "System Tools"
>
>* Click on the "Done" button to return
>```

>9. Click on `Installation Destination`, set the following:
>
>```json
>* Use "Custom" Storage Configuration
>* Click on the "Done" button to continue
>* Configure using below Partition Table
>* Click on the "Done" button to continue
>* Click on the "Accept Changes" button to continue
>```

>### Partition Table
>
>:::note
>
>This partition table is based on the requirements of the NIST security profile. It is the recommended layout even if the profile is not enabled to ensure future compatibility with the security profiles. Adjust the sizes of the partions as needed.
>
>:::
>
>:::important
>
>The NFS Server is intended to be deployed on a bare-metal system. The OS partitions should be deployed in a RAID-1 (Mirrored) volume and the storage used to host NFS mounts should be configured as a RAID-6 volume.
>
>:::
>
>| Partition Name   | Size  | Device Type        | Drives       | File System            |
>|------------------|-------|-------------------|--------------|------------------------|
>| /boot/efi        | 2 GiB | Standard Partition | OS (RAID-1)  | EFI System Partition   |
>| /boot            | 2 GiB | Standard Partition | OS (RAID-1)  | xfs                    |
>| /                | 32 GiB| LVM                | OS (RAID-1)  | xfs                    |
>| /home            | 2 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /srv/nfs/app     | 10 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /srv/nfs/backup  | 10 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /srv/nfs/home    | 20 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /srv/nfs/mirror  | 2 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /srv/nfs/scratch | 10 TiB| LVM                | Data (RAID-6)| xfs                    |
>| /tmp             | 8 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var             | 32 GiB| LVM                | OS (RAID-1)  | xfs                    |
>| /var/log         | 5 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var/log/audit   | 5 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| /var/tmp         | 8 GiB | LVM                | OS (RAID-1)  | xfs                    |
>| swap             | 2 GiB | LVM                | OS (RAID-1)  | swap                   |

>10. Click on `KDUMP`, set the following:
>
>```json
>* Uncheck "Enable kdump"
>* Click on the "Done" button to return
>```

>11. Click on `Root Password`, set the following:
>
>:::note
>
>Only a root account will be created
>
>:::
>
>```json
>* Set a strong Root password
>* Click on the "Done" button to return
>```

>12. Click on the `Begin Installation` button to begin

>13. Click on the `Reboot System` button when installation has completed