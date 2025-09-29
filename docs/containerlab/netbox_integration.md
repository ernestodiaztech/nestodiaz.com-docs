---
sidebar_position: 2
---

# 2. NetBox DCIM/IPAM Integration

NetBox is the industry-standard DCIM (Data Center Infrastructure Management) and IPAM (IP Address Management) tool used in enterprise environments. It serves as the "source of truth" for network infrastructure data.

| Server | IP | Service | Purpose |
|----------|----------|----------|----------|
| ContainerLab   | 10.33.99.12   | Network Lab   | Virtual network devices   |
| Ansible   | 10.33.99.13   | Automation   | Configuration management   |
| Netbox   | 10.33.99.14   | DCIM/IPAM   | Source of truth   |

## NetBox Installation

First, I updated the system.

```bash
sudo apt update && sudo apt upgrade -y
```

Then installed PostgreSQL

```bash
sudo apt install -y postresql
```

Then verified that PostgreSQL 14 or later was installed.

```bash
psql -V
```

Now I needed to create the database.

I started by invoking the PostgreSQL shell as the system Postgres user.

```bash
sudo -u postgres psql
```

I then created the database, user, and made the user the owner of the database.

```bash
CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD 'HardtoGuess123!';
ALTER DATABASE netbox OWNER TO netbox;
\q
```

:::info[Verify Service Status]

```bash
$ psql --username netbox --password --host localhost netbox
Password for user netbox: 
psql (12.5 (Ubuntu 12.5-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```

:::

I then moved onto installing Redis.

```bash
sudo apt install -y redis-server
```

I verified the version is at least 4.0

```bash
redis-server -v
```

