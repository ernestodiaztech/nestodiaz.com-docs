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

### Prerequisites

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

:note[Command Explanation]

- `CREATE DATABASE netbox;` - Creates a new database named "netbox"
- `CREATE USER netbox WITH PASSWORD 'HardtoGuess123!';` - Creates a database user named "netbox" with the specified password
- `ALTER DATABASE netbox OWNER TO netbox;` - Makes the "netbox" user the owner of the "netbox" database (grants full privileges)
- `\q` - PostgreSQL command to quit the interactive shell

:::

:::info[Verify Database Connection]

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

### Installation

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

### Configuration

Next, I moved into the NetBox configuration direcotry and made a copy of `configuration_example.py`, and named it `configuration.py`.

```bash
cd /opt/netbox/netbox/netbox/
sudo cp configuration_example.py configuration.py
```

I then edited the file.

```bash
sudo nano configuration.py
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

SECRET_KEY = 'Pvp_F0sdf2ryry9r5PC211y4j8j98cL8hcJ5sdf595954848156df5sdf657'
```

:::info[Configuration File Explained]

**ALLOWED_HOSTS**

- `10.33.99.14` - NetBox server IP
- `localhost`, `127.0.0.1` - Local access for testing

**DATABASE**

- `ENGINE` - Database backend (PostgreSQL)
- `NAME` - Database name (must match what we created earlier)
- `USER` - Database username
- `PASSWORD` - Database password (change this in production!)
- `HOST` - Database server location (localhost means same machine)
- `PORT` - Leave empty to use default PostgreSQL port (5432)
- `CONN_MAX_AGE` - Keep database connections alive for 300 seconds (improves performance)

**REGIS**

- NetBox uses two separate Redis databases
- `tasks` (database 0) - Background job queue
- `caching` (database 1) - Application caching
- `PORT: 6379` - Default Redis port
- Separating tasks and caching improves performance and prevents conflicts

:::

To create the secret key, I ran the following script provided by NetBox.

```bash
python3 ../generate_secret_key.py
```

Now I was able to finally install NetBox by running the upgrade script.

```bash
sudo /opt/netbox/upgrade.sh
```

:::note

This script automates several tasks:

1. Creates a Python virtual environment in `/opt/netbox/venv/`
2. Installs all Python dependencies from `requirements.txt`
3. Runs database migrations (creates tables and schema)
4. Collects static files (CSS, JavaScript) for web serving
5. Clears expired sessions from the database

The virtual environment isolates NetBox's Python dependencies from system packages.

:::

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

### Development Server

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

### Gunicorn

I opted to use Gunicorn instead of UWSGI.

:::info[Why Gunicorn]

Gunicorn is chosen over uWSGI for its simplicity, active maintenance, and better integration with modern Python applications. It's the recommended WSGI server in the official NetBox documentation.

:::

I copied the default configuration file.

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

### HTTP Server

Now to setup the HTTP server.

I generated my own self-signed certificate.

:::info[Why Self-Signed]

For internal lab environments not exposed to the internet, self-signed certificates provide encryption without the overhead of managing CA-signed certificates. In production environments, use proper SSL certificates from a trusted Certificate Authority.

:::

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

:::note[Command Explained]

**Server Block**

- `listen 80;` - Listen on port 80 (HTTP)
- `server_name 10.33.99.14;` - Respond to requests for this IP address

- `client_max_body_size 25m;` - Maximum upload size (25 megabytes)

- `location /static/`

  - `alias /opt/netbox/netbox/static/;` - Serve static files (CSS, JS, images) directly from filesystem
  - Nginx handles static files efficiently without involving the Python application
  - The trailing slashes are important for proper path resolution

- `proxy_set_header X-Forwarded-Host $http_host;` - Passes original Host header to backend. Tells NetBox what hostname client used.
- `proxy_set_header X-Real-IP $remote_addr;` - Passes client's real IP address. Without this, NetBox would only see 127.0.0.1.
- `proxy_set_header X-Forwarded-Proto $scheme;` - Passes protocol (http/https). Helps NetBox generate correct URLs.
- `add_header P3P ...` - Platform for Privacy Preferences header. Legacy compatibility for older browsers

:::

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

## Ansible Integration with NetBox

First, I installed Netbox Ansible collection.

```bash
source ~/ansible-env/ansible-venv/bin/activate
pip install pynetbox
pip install pytz
ansible-galaxy collection install netbox.netbox
```

I then needed to configure NetBox API access.

Back in NetBox's web interface I went to `Admin` > `API Tokens`.

- Created a new token for user `netbox-automation`
- Copied the token value

### Inventory Configuration

I then created an inventory configuration file.

```bash
mkdir -p ~/network-automation/inventory_plugins
cd ~/network-automation
sudo nano inventory/netbox_inventory.yml
```

```yaml title="netbox_inventory.yml"
---
plugin: netbox.netbox.nb_inventory
api_endpoint: http://10.33.99.14
token: your_api_token_here
validate_certs: False
config_context: False
interfaces: True
group_by:
  - device_roles
  - sites
  - device_types
query_filters:
  - site: nesto-diaz.com
compose:
  ansible_host: primary_ip4
keyed_groups:
  - key: device_role.slug
    prefix: role
  - key: site.slug
    prefix: site
```

:::note[Inventory Configuration Explanation]

- `plugin: netbox.netbox.nb_inventory` - Specifies the NetBox dynamic inventory plugin
- `api_endpoint: http://10.33.99.14` - NetBox API URL
- `token: your_api_token_here` - Replace with your actual API token
- `validate_certs: False` - Skip SSL certificate validation (OK for self-signed certs in lab)
  - Set to `True` in production with proper certificates
- `config_context: False` - Don't fetch config context data (improves performance)
- `interfaces: True` - Include interface data in inventory

group_by:

- Creates Ansible groups based on NetBox attributes
- `device_roles` - Groups like "router", "switch", "host"
- `sites` - Groups by site name
- `device_types` - Groups by device model

query_filters:

- `site: nesto-diaz-com` - Only fetch devices from this site
- Reduces API calls and inventory size
- Can filter by role, status, tags, etc.

compose:

- `ansible_host: primary_ip4` - Use device's primary IPv4 as connection target
- NetBox's primary IP becomes Ansible's management IP

keyed_groups:

- Creates groups with prefixes for better organization
- `role_router` instead of just `router`
- `site_nesto-diaz-com` instead of just `nesto-diaz-com`
- Prevents group name collisions
- :::

I then tested the playbook.

```bash
export DOCKER_HOST=ssh://nesto@10.33.99.12
ansible-inventory -i inventory/netbox_inventory.yml --list
```

:::note[Command Explanation]

- `export DOCKER_HOST=ssh://nesto@10.33.99.12` - Sets environment variable for Docker connections
  - Tells Ansible to connect to Docker on the ContainerLab server
  - Uses SSH tunnel to reach containers
- `ansible-inventory` - Ansible command to view inventory
- `-i inventory/netbox_inventory.yml` - Specifies inventory file to use
- `--list` - Outputs full inventory in JSON format
  - Shows all hosts, groups, and variables
  - Alternative: `--graph` for tree view, `--host <hostname>` for single host

:::

Example Output (shortened)

```bash
    "device_roles_host": {
        "hosts": [
            "host1"
        ]
    },
    "device_roles_router": {
        "hosts": [
            "r1",
            "r2"
        ]
    },
    "device_roles_switch": {
        "hosts": [
            "sw`"
        ]
    },
    "device_types_alpine-host": {
        "hosts": [
            "host1"
        ]
    },
    "device_types_alpine-switch": {
        "hosts": [
            "sw`"
        ]
    },
    "device_types_frr-router": {
        "hosts": [
            "r1",
            "r2"
        ]
    },
    "site_lab_network": {
        "hosts": [
            "host1",
            "r1",
            "r2",
            "sw`"
        ]
    },
    "sites_lab-network": {
        "hosts": [
            "host1",
            "r1",
            "r2",
            "sw`"
        ]
    }
}
```

### NetBox-Driven Configuration

I then created a dynamic playbook.

:::info[NetBox-Driven Configuration Playbook Explanation]

In enterprise networks, NetBox serves as the "source of truth" - meaning:

- All network design decisions are documented in NetBox FIRST
- Automation pulls data FROM NetBox to configure devices
- Devices don't dictate their own configuration; NetBox does

**Traditional Approach**

```bash
Network Engineer → Manually configures Router → Documents it later (maybe)
```

**NetBox-Driven Approach**

```bash
Network Engineer → Updates NetBox → Automation configures Router → Router matches NetBox
```

:::

```yaml title="playbooks/netbox-driven-config.yml"
---
- name: Configure devices based on NetBox data
  hosts: all
  gather_facts: no
  vars:
    netbox_url: "http://10.33.99.14"
    netbox_token: "a21c98f5sdfs3c227fd233sdf9sdfsd04060c6a046"

  tasks:
    - name: Get device data from NetBox
      uri:
        url: "{{ netbox_url }}/api/dcim/devices/"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
        return_content: yes
      delegate_to: localhost
      run_once: true
      register: netbox_devices

    - name: Configure router interfaces based on NetBox data
      ansible.builtin.shell: |
        vtysh -c 'configure terminal' \
              -c 'interface {{ item.name }}' \
              -c 'description {{ item.description | default("Managed by NetBox") }}' \
              -c 'ip address {{ item.ip_address }}' \
              -c 'exit' \
              -c 'write'
      when: "'router' in group_names"
      loop: "{{ device_interfaces | default([]) }}"

    - name: Verify configuration
      ansible.builtin.shell: vtysh -c "show interface brief"
      when: "'router' in group_names"
      register: interface_status

    - name: Display interface status
      debug:
        var: interface_status.stdout_lines
      when: "'router' in group_names"
```

<details>
  <summary>Playbook Explanation</summary>
:::note

**Playbook Header**

- `hosts: all` - Run on ALL devices from your inventory (r1, r2, sw1, host1)
- `gather_facts: no` - Skip collecting system info (faster startup)
- `vars:` - Define variables used throughout the playbook
  - `netbox_url` - Where NetBox lives (your NetBox server)
  - `netbox_token` - API authentication token (like a password for API access)

**Task 1: Fetching Data from NetBox API**

What This Does: Makes an HTTP API call to NetBox to retrieve ALL device information.

- `uri:` - Ansible's module for making HTTP/HTTPS requests (like `curl` but in Ansible)
- `url: "{{ netbox_url }}/api/dcim/devices/"`
  - Constructs the full URL: `http://10.33.99.14/api/dcim/devices/`
  - This API endpoint returns JSON data about all devices in NetBox
  - You can test this manually: curl `http://10.33.99.14/api/dcim/devices/`
- `method: GET`
  - HTTP GET request (retrieving data, not changing anything)
  - Other methods: POST (create), PUT/PATCH (update), DELETE (remove)
- `headers:`
  - HTTP headers are metadata sent with the request
  - NetBox requires authentication via an API token
  - Format: `Token 65sdf95sdfw9erqqff`
  - This is like showing your ID badge to get into a building
- `return_content: yes`
  - Store the API response body (the JSON data NetBox sends back)
  - Without this, you'd only get HTTP status code (200, 404, etc.)
- `delegate_to: localhost`
  - CRITICAL CONCEPT: Run this task on the Ansible control machine, NOT on the target devices
  - Why? Your routers can't make API calls to NetBox, but your Ansible server can
  - Think of it like: "Hey Ansible server, YOU make this API call, not the router"
- `run_once: true`
  - Even though this playbook runs against multiple devices (r1, r2, sw1, host1), only fetch NetBox data ONCE
  - Without this, Ansible would fetch the same data 4 times (wasteful!)
  - The data is shared across all hosts
- `register: netbox_devices`
  - Store the API response in a variable called `netbox_devices`
  - Now you can use `{{ netbox_devices }}` in later tasks
  - Contains: status code, headers, JSON content, etc.

**Task 2: Configure Interfaces Based on NetBox Data**

1. `ansible.builtin.shell: |`

- Run shell commands on the target device
- The | means multi-line string follows (everything indented below)

2. `vtysh -c 'configure terminal' \`

- `vtysh` - FRRouting's CLI interface (like Cisco IOS CLI)
- `-c` - Execute a command
- `'configure terminal'` - Enter configuration mode
- `\` - Line continuation (command continues on next line)

3. `-c 'interface {{ item.name }}'`

- Configure a specific interface
- `{{ item.name }}` - Jinja2 template variable
- Example: If `item.name = "eth1"`, this becomes `interface eth1`
- `item` comes from the `loop:` directive (explained below)

4. `-c 'description {{ item.description | default("Managed by NetBox") }}'`

- Set interface description
- `{{ item.description }}` - Description from NetBox
- `| default("Managed by NetBox")` - Jinja2 filter: If no description in NetBox, use this default text
- Example: `description Connection to Switch 1` or `description Managed by NetBox`

5. `-c 'ip address {{ item.ip_address }}'`

- Assign IP address to interface
- Example: `ip address 10.1.1.1/24`
- This comes directly from NetBox's interface IP assignment

6. `-c 'exit' (first one)`

- Exit interface configuration mode

7. `-c 'exit' (second one)`

- Exit global configuration mode

8. `-c 'write'`

- Save configuration to startup-config
- Without this, config is lost on reboot!

9. `when: "'router' in group_names"`

- Conditional execution: Only run on devices in the "router" group
- `group_names` - List of groups this host belongs to (from inventory)
- Example: r1 and r2 are routers, so this runs. sw1 and host1 are NOT routers, so this skips.
- This prevents trying to run `vtysh` on Alpine Linux hosts (which don't have it)

10. `loop: "{{ device_interfaces | default([]) }}"`

- IMPORTANT: Iterate over a list of interfaces
- `device_interfaces` - Should be a variable containing interface data
- `| default([])` - If `device_interfaces` doesn't exist, use empty list (prevents errors)
- Each iteration, `item` contains one interface's data

:::

</details>

Then I ran the playbook.

```bash
ansible-playbook -i inventory/netbox_inventory.yml playbooks/netbox-driven-config.yml
```

:::note[Example Output]

```bash
(ansible-venv) nesto@ansible:~/network-automation$ ansible-playbook -i inventory/netbox_inventory.yml playbooks/netbox-driven-config.yml
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Configure devices based on NetBox data] ****************************************************************************************************************************************************************
[WARNING]: Found variable using reserved name: serial
[WARNING]: Found variable using reserved name: tags

TASK [Get device data from NetBox] ***************************************************************************************************************************************************************************
ok: [host1 -> localhost]

TASK [Configure router interfaces based on NetBox data] ******************************************************************************************************************************************************
skipping: [host1]
skipping: [r2]
skipping: [r1]
skipping: [sw`]

TASK [Verify configuration] **********************************************************************************************************************************************************************************
skipping: [host1]
skipping: [r1]
skipping: [r2]
skipping: [sw`]

TASK [Display interface status] ******************************************************************************************************************************************************************************
skipping: [host1]
skipping: [r1]
skipping: [r2]
skipping: [sw`]

PLAY RECAP ***************************************************************************************************************************************************************************************************
host1                      : ok=1    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
r1                         : ok=0    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
r2                         : ok=0    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
sw`                        : ok=0    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

:::

### Advanced NetBox Integration

NetBox supports Jinja2 configuration templates for generating device configs.

I went to `Provisioning` > `Config Templates` and crated a template.

```bash
! Configuration generated from NetBox
! Device: {{ device.name }}
! Site: {{ device.site.name }}
!
hostname {{ device.name }}
!
{% for interface in device.interfaces.all %}
interface {{ interface.name }}
 description {{ interface.description | default("Interface managed by NetBox") }}
 {% if interface.ip_addresses.first %}
 ip address {{ interface.ip_addresses.first.address }}
 {% endif %}
 no shutdown
!
{% endfor %}
!
router ospf
 {% for prefix in site_prefixes %}
 network {{ prefix.prefix }} area {{ prefix.custom_fields.ospf_area | default(0) }}
 {% endfor %}
!
```

Then I added some custom fields for network automation.

I went to `Customization` > `Custom Fields`

I created:

- OSPF Area (for prefixes)
- VLAN ID (for interfaces)
- BGP ASN (for devices)

I then setup webhooks to trigger automation on NetBox changes.

I went to `Operations` > `Webhooks`.

I created the following webhook:

- Name: Device Change Webhook
- Content Types: dcim.device
- URL: `http://10.33.99.13:9000/netbox-webhook`
- Evemts" Created. Updated. Deleted

I also created a validation playbook.

```bash
---
- name: Validate NetBox data against actual devices
  hosts: all
  gather_facts: no
  tasks:
    - name: Get actual device configuration
      ansible.builtin.shell: vtysh -c "show running-config"
      register: actual_config
      when: "'router' in group_names"

    - name: Compare with NetBox expected config
      debug:
        msg: "Configuration validation needed"
      # Add validation logic here

    - name: Report discrepancies to NetBox
      uri:
        url: "{{ netbox_url }}/api/extras/journal-entries/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          assigned_object_type: "dcim.device"
          assigned_object_id: "{{ device_id }}"
          kind: "warning"
          comments: "Configuration drift detected"
      when: config_drift_detected | default(false)
```

And an automated backups.

```bash
---
- name: Backup configurations to NetBox
  hosts: routers
  tasks:
    - name: Get running configuration
      ansible.builtin.shell: vtysh -c "show running-config"
      register: running_config

    - name: Upload config to NetBox
      uri:
        url: "{{ netbox_url }}/api/extras/config-contexts/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
        body_format: json
        body:
          name: "{{ inventory_hostname }}-backup-{{ ansible_date_time.epoch }}"
          data:
            backup_config: "{{ running_config.stdout }}"
            backup_date: "{{ ansible_date_time.iso8601 }}"
```
