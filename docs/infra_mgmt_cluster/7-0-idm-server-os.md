---
sidebar_position: 7
---

# 7. IdM Server OS Installation

## Configuration

<table>
  <thead>
    <tr>
      <th colspan="2">IdM Server</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>OS</strong></td>
      <td>Red Hat Enterprise Linux 8</td>
    </tr>
    <tr>
      <td><strong>Installation Type</strong></td>
      <td>Virtual Machine Guest</td>
    </tr>
    <tr>
      <td><strong>vCPU</strong></td>
      <td>4</td>
    </tr>
    <tr>
      <td><strong>Memory</strong></td>
      <td>16 GB</td>
    </tr>
    <tr>
      <td><strong>Storage</strong></td>
      <td>96 GB (qcow2)</td>
    </tr>
    <tr>
      <td><strong>Network Interface</strong></td>
      <td>Unique/Bridged IPv4</td>
    </tr>
    <tr>
      <td><strong>Hostname</strong></td>
      <td>idm</td>
    </tr>
    <tr>
      <td><strong>FQDN</strong></td>
      <td>idm.nestodiaz.com</td>
    </tr>
  </tbody>
</table>

## OS Installation Steps

1. Boot VM using Red Hat Enterprise Linux (RHEL) 8.8 ISO
2. Select `Install Red Hat Enterprise Linux 8.8` and press `ENTER`
3. Select your language to use during the installation process and press `Continue`

:::note

You should now see the `Installation Summary` GUI.

:::

4. Click on `Language Support`, set the following:

```json
* Select your language to be used following installation
* Click on the "Done" button to return
```

5. Click on `Network & Host name`, set the following:

```json
* Host Name to: idm
* Click on the "Configure..." button

    1. Under the General tab, Check "Connect automatically with priority"
    2. Under the Ethernet tab, Set Link negotiation to "Automatic"
    3. Under the IPv4 Settings tab,

        * Select Method: "Manual"
        * Set IP Address and DNS Servers

    4. Under IPv6 Settings tab,

        * Select Method "Link-Local Only"

    5. Click on the "Save" button

* Turn Ethernet (ens18) "On"
* Hit the "Apply" button to set the host name
* Click on the "Done" button to return
```

6. Click on `Time & Date`, set the following:

```json
* Select your timezone
* Enable "Network Time"
* Configure NTP servers as needed
* Click on the "Done" button to return
```

7. Click on `Installation Source`, set the following:

```json
* Click on "Auto-detected installation media"
* Click on the "Done" button to return
```

8. Click on `Software Selection`, set the following:

```json
* Select Base Environment: "Server"

    1. Check: "Guest Agents"
    2. Check: "Network File System Client"
    3. Check: "Security Tools"
    4. Check: "System Tools"

* Click on the "Done" button to return
```

9. Click on `Installation Destination`, set the following:

```json
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
>This partition table is based on the requirements of the NIST security profile. It is the recommended layout even if the profile is not enabled to ensure future compatibility >with the security profiles.
>
>:::
>
><table>
>  <thead>
>    <tr>
>      <th colspan="1">Partition Name</th>
>      <th colspan="1">Size</th>
>      <th colspan="1">Device Type</th>
>      <th colspan="1">File System</th>
>    </tr>
>  </thead>
>  <tbody>
>    <tr>
>      <td><strong>/boot</strong></td>
>      <td>2 GB</td>
>      <td><strong>Standard Partition</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/</strong></td>
>      <td>32 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/tmp</strong></td>
>      <td>8 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/var</strong></td>
>      <td>32 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/var/log</strong></td>
>      <td>5 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/var/log/audit</strong></td>
>      <td>5 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>/var/tmp</strong></td>
>      <td>8 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>xfs</td>
>    </tr>
>    <tr>
>      <td><strong>swap</strong></td>
>      <td>2 GB</td>
>      <td><strong>LVM</strong></td>
>      <td>swap</td>
>    </tr>
>  </tbody>
></table>

10. Click on `KDUMP`, set the following:

```json
* Uncheck "Enable kdump"
* Click on the "Done" button to return
```

11. Click on `Root Password`, set the following:

```json
* Set a strong Root password
* Click on the "Done" button to return
```

12. Click on the `Begin Installation` button to begin
13. Click on the `Reboot System` button when installation has completed