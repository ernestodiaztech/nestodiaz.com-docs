---
sidebar_position: 2
---

# 2. NetBox DCIM/IPAM Integration

NetBox is the industry-standard DCIM (Data Center Infrastructure Management) and IPAM (IP Address Management) tool used in enterprise environments. It serves as the "source of truth" for network infrastructure data.

| Server       | IP          | Service     | Purpose                  |
| ------------ | ----------- | ----------- | ------------------------ |
| ContainerLab | 10.33.99.12 | Network Lab | Virtual network devices  |
| Ansible      | 10.33.99.13 | Automation  | Configuration management |
| Netbox       | 10.33.99.14 | DCIM/IPAM   | Source of truth          |

## NetBox Installation

This section will be almost an exact copy of the official installation guide (https://netboxlabs.com/docs/netbox/installation/). The only changes I made were in the Nginx configuration file.

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

:::warning

NetBox requires PostgreSQL 14 or later. Please note that MySQL and other relational databases are not supported.

:::

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

Before installing NetBox, I installed the required system packages.

```bash
sudo apt install -y python3 python3-pip python3-venv python3-dev \
build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev \
libssl-dev zlib1g-dev
```

Then I verified my installed Python version is at least 3.10.

```bash
python3 -V
```

Then, I cloned the Git Repository, because it will be easier for future upgrades.

First, I made the `opt/netbox` directory.

```bash
sudo mkdir -p /opt/netbox/
cd /opt/netbox/
```

Then cloned the repo.

```bash
sudo git clone https://github.com/netbox-community/netbox.git .
```

Then I checked the release page for the current version (https://github.com/netbox-community/netbox/releases).

I then checked out the version 4.4.1.

```bash
sudo git checkout v4.4.1
```

Then I created a NetBox system user, which I will use to run WSGI and HTTP.

```bash
sudo adduser --system --group netbox
```

Then assigned ownership of the media directories.

```bash
sudo chown --recursive netbox /opt/netbox/netbox/media/
sudo chown --recursive netbox /opt/netbox/netbox/reports/
sudo chown --recursive netbox /opt/netbox/netbox/scripts/
```

Next, I moved into the NetBox configuration direcotry and made a copy of `configuration_example.py`, and named it `configuration.py`.

```bash
cd /opt/netbox/netbox/netbox/
sudo cp configuration_example.py configuration.py
```

```bash title="configuration.py"
ALLOWED_HOSTS = ['10.33.99.14', 'localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'netbox',
        'USER': 'netbox',
        'PASSWORD': 'HardtoGuess123!',
        'HOST': 'localhost',
        'PORT': '',
        'CONN_MAX_AGE': 300,
    }
}

REDIS = {
    'tasks': {
        'HOST': 'localhost',
        'PORT': 6379,
        'USERNAME': '',
        'PASSWORD': '',
        'DATABASE': 0,
        'SSL': False,
    },
    'caching': {
        'HOST': 'localhost',
        'PORT': 6379,
        'USERNAME': '',
        'PASSWORD': '',
        'DATABASE': 1,
        'SSL': False,
    }
}

SECRET_KEY = 'Pvp_F0sdf284sBxQ9PCEjd-uKcL8hcJ5sdf5959sdf5sdf657'
```

To create the secret key, I ran the following command:

```bash
python3 ../generate_secret_key.py
```

Now I was able to finally install NetBox by running the upgrade script.

```bash
sudo /opt/netbox/upgrade.sh
```

Now to create the superuser for NetBox.

First, I entered the Python virtual environment that was created by the upgrade script.

```bash
source /opt/netbox/venv/bin/activate
```

Then ran the following command:

```bash
cd /opt/netbox/netbox
python3 manage.py createsuperuser
```

I now tested to see if NetBox was installed correctly by running NetBox's development server.

```bash
python3 manage.py runserver 0.0.0.0:8000 --insecure
```

:::note[Example Output]

```bash
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
August 30, 2021 - 18:02:23
Django version 3.2.6, using settings 'netbox.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

:::

I then opened a browser and went to http:10.33.99.14 to verify the NetBox login screen appeared.

I then shutdown the development server by typing `CTRL+C`.

I opted to use Gunicorn instead of UWSGI. So, I copied the default configuration file.

```bash
sudo cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
```

Systemd will be used to control both Gunicorn and NetBox's background worker process.

I then copied `contrib/netbox.service` and `contrib/netbox-rq.service` to the `/etc/systemd/system/` directory and reloaded the systemd daemon.

```bash
sudo cp -v /opt/netbox/contrib/*.service /etc/systemd/system/
sudo systemctl daemon-reload
```

Then started the `netbox` and `netbox-rq` services and enabled them to start at boot.

```bash
sudo systemctl enable --now netbox netbox-rq
```

To verify WSGI service is running, I ran the following:

```bash
systemctl status netbox.service
```

:::note[Example Output]

```bash
netbox.service - NetBox WSGI Service
     Loaded: loaded (/etc/systemd/system/netbox.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-08-30 04:02:36 UTC; 14h ago
       Docs: https://docs.netbox.dev/
   Main PID: 1140492 (gunicorn)
      Tasks: 19 (limit: 4683)
     Memory: 666.2M
     CGroup: /system.slice/netbox.service
             ├─1140492 /opt/netbox/venv/bin/python3 /opt/netbox/venv/bin/gunicorn --pid /va>
             ├─1140513 /opt/netbox/venv/bin/python3 /opt/netbox/venv/bin/gunicorn --pid /va>
             ├─1140514 /opt/netbox/venv/bin/python3 /opt/netbox/venv/bin/gunicorn --pid /va>
```

:::

Now to setup the HTTP server.

I generated my own self-signed certificate, since I won't be exposing NetBox to the internet.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/netbox.key \
-out /etc/ssl/certs/netbox.crt
```

Then I installed Nginx.

```bash
sudo apt install -y nginx
```

I then copied the Nginx configuration file provided by NetBox to `/etc/nginx/sites-available/netbox`.

```bash
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
```

This is where I changed the Nginx configuration.

```bash
sudo nano /etc/nginx/sites-available/netbox
```

```bash title="netbox"
server {
    listen 80;
    server_name 10.33.99.14;

    client_max_body_size 25m;

    location /static/ {
        alias /opt/netbox/netbox/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
    }
}
```

Then restarted Nginx

```bash
sudo systemctl restart nginx
```

Now Netbox is available on http://10.33.99.14

## NetBox Configuration

**Create Site**

I went to `Organization` > `Sites`

- Added a new site
  - Name: NestoDiaz.com
  - Slug: nesto-diaz.com
  - Status: Active

**Create Rack**

I went to `Racks` > `Rack`

- Added a new rack
  - Site: NestoDiaz.com
  - Name: Virtual Rack 1
  - Status: Active

**Create Device Roles**

I went to `Devices` > `Device Roles`

- Added new device roles
  - Router (color: blue)
  - Switch (color: green)
  - Host (color: grey

**Create Manufacturer**

I went to `Devices` > `Manufacturers`

- Added new manufacturers
  - FRRouting
  - Alpine

**Create Device Types**

I went to `Devices` > `Device Types`

- Added new device types
  - FRR Router (manufacturer: FRRouting)
  - Alpine Switch (manufacturer: Alpine)
  - Alpine Host (manufacturer: Alpine)

**Add Network Devices**

I went to `Devices` > `Devices`

insert table

**Create VRFs**

I went to `IPAM` > `VRFs`

- Added new VRF
  - Name: default
  - Router Distinguisher: 65000:1

**Create IP Prefixes**

I went to `IPAM` > `Prefixes`

- Added new prefixes
  - 172.20.20.0/24 (Management Network)
  - 10.1.1.0/24 (Router 1 LAN)
  - 10.1.2.0/24 (Router 2 LAN)
  - 10.1.100.0/24 (Host Network)

**Assign IP Addresses**

I went to `IPAM` > `IP Addresses`

- Added IP addresses for each device
  - r1: 172.20.20.11/24, 10.1.1.1/24
  - r2: 172.20.20.12/24, 10.1.2.1/24
  - sw1: 172.20.20.13/24
  - host1: 172.20.20.14/24, 10.1.100.1/24