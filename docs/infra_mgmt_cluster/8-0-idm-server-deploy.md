---
sidebar_position: 8
---

# 8. IdM Server Deployment

## Post Deployment

>The IdM Management Dashboard can be accessed at the following URL:  
>https://idm.nestodiaz.com
>
>Username: `admin`  
>Password: `ADMIN_PASSWORD`

## Deployment Steps

The IdM server can either be deployed as a standalone VM or on the NFS server. IdM and NFS are both required for users to log in and deploying both on the same server reduces the number of required systems to be online before users can log in.

This guide is writen for a standalone VM instance.

:::note

Instructions assume execution using the `root` account.

:::

>1. Connect the system to the NFS server:

>2. Install the Random Number Generator Service
>
>```json
>dnf -y install rng-tools
>systemctl enable --now rngd
>```

>3. Set the hostname to a FQDN:
>
>```json
>hostname set-hostname idm.nestodiaz.com
>```

>4. Ensure the timezone is properly set:
>
>```json
>timedatectl set-timezone America/New_York
>timedatectl set-local-rtc
>```

>5. `Optional` Add hostname to `/etc/hosts` if DNS does not resolve the FQDN:
>
>:::note
>
>If synchronizing hosts files across systems ensure the FQDN name is before any short names.
>
>`IP_ADDRESS HOST_FQDN HOST_SHORTNAME`
>
>:::
>
>```json
>sh -c `cat >> /etc/hosts <<EOL
>
>IP_ADDRESS HOST_FQDN HOST_SHORTNAME
>
>EOL`
>```

>6. Verify hostname resolves to an IPv4 address that is not equal to the loopback address:
>
>```json
>dig +short idm.nestodiaz.com A
>```

>7. Verify reverse DNS configuration (PTR records):
>
>```json
>dig +short -x 192.168.1.80
>```

>8. Install Chrony NTP server:
>
>```json
>dnf -y install chrony
>```

>9. Configure Chrony to allow remote clients:
>
>```json
>sed -i 's@#allow 192.168.0.0/16@allow 192.168.1.0/24@g' /etc/chrony.conf
>```

>10. Restart chrondy service
>
>```json
>systemctl restart chronyd
>```

>11. Verify IdM module information:
>
>```json
>dnf module info idm:DL1
>```

>12. Enable the idm:DL1 stream and sync repositories:
>
>```json
>dnf module -y enable idm:DL1
>dnf distro-sync
>```

>13. Install IdM Server module without an integrated DNS:
>
>```json
>dnf module -y install idm:DL1/server
>```

>14. Suppress Negotiate Headers:
>
>```json
>mkdir -p /etc/httpd/conf.d/
>cat > /etc/httpd/conf.d/gssapi.conf <<EOF
>BrowserMatch Windows gssapi-no-negotiate
>EOF
>```

>15. Install IdM Server:
>
>```json
>ipa-server-install \
>  --domain=engwsc.example.com \
>  --realm=ENGWSC.EXAMPLE.COM \
>  --ds-password=DM_PASSWORD \
>  --admin-password=ADMIN_PASSWORD \
>  --unattended
>```

>16. Configure firewalld rules:
>
>```json
>systemctl enable --now firewalld
>firewall-cmd --zone=public --add-source=192.168.1.0/24 --permanent
>firewall-cmd --zone=public --add-service={http,https,ntp,freeipa-4} --permanent
>firewall-cmd --reload
>```