---
sidebar_position: 1
---

# 1. ContainerLab Setup and Ansible Integration

Here I walk you through on how I set up ContainerLab and Ansible to build and automate containerized network labs. It covers installing the required tools, creating and testing a simple FRRouting-based topology, configuring Ansible with Docker-based connections, and running playbooks for connectivity checks and basic network configuration.

## Network Overview

| Server       | IP          | Subnet        | Role                    |
| ------------ | ----------- | ------------- | ----------------------- |
| ContainerLab | 10.33.99.12 | 255.255.255.0 | Runs network containers |
| Ansible      | 10.33.99.13 | 255.255.255.0 | Automation control node |

## ContainerLab Installation

### Prerequisites

First, I updated the system.

```bash
sudo apt update && sudo apt upgrade -y
```

Then I installed Docker and Docker Compose

```bash
sudo apt install -y docker.io docker-compose
```

Then enabled Docker to start on boot, and started the service.

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Then added my current user to the Docker group.

```bash
sudo usermod -aG docker $USER
```

I then applied the group membership without logging out.

```bash
newgrp docker
```

:::warning

After running `newgrp docker`, you need to use it in the current terminal session. For permanent effect across all future sessions, log out and log back in.

:::

### Install ContainerLab

Then installed Containerlab.

```bash
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

To verify everything installed correctly I ran:

```bash
containerlab version
```

It should show something like:

```bash
  ____ ___  _   _ _____  _    ___ _   _ _____ ____  _       _
 / ___/ _ \| \ | |_   _|/ \  |_ _| \ | | ____|  _ \| | __ _| |__
| |  | | | |  \| | | | / _ \  | ||  \| |  _| | |_) | |/ _` | '_ \
| |__| |_| | |\  | | |/ ___ \ | || |\  | |___|  _ <| | (_| | |_) |
 \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\_|\__,_|_.__/

    version: 0.70.1
     commit: 3828f87c
       date: 2025-09-24T10:04:31Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.70/#0701
```

## Testing

### Basic Topology

Next, I created a simple topology for testing.

I made a directory for my topologies.

```bash
mkdir -p ~/containerlab/topologies
cd ~/containerlab/topologies
```

Then made a basic network topology.

```yaml title="basic-network.clab.yml"
name: basic-network

topology:
  nodes:
    r1:
      kind: linux
      image: frrouting/frr:latest
      mgmt-ipv4: 172.20.20.11
      exec:
        - sysctl -w net.ipv4.ip_forward=1
        - sysctl -w net.ipv6.conf.all.forwarding=1
        - sleep 5
        - sed -i "s/ospfd=no/ospfd=yes/" /etc/frr/daemons
        - /usr/lib/frr/ospfd -d
    r2:
      kind: linux
      image: frrouting/frr:latest
      mgmt-ipv4: 172.20.20.12
      exec:
        - sysctl -w net.ipv4.ip_forward=1
        - sysctl -w net.ipv6.conf.all.forwarding=1
        - sleep 5
        - sed -i "s/ospfd=no/ospfd=yes/" /etc/frr/daemons
        - /usr/lib/frr/ospfd -d
    sw1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.13
      exec:
        - ip link add br0 type bridge
        - ip link set br0 up
        - sleep 2
        - for iface in eth1 eth2 eth3; do if ip link show $iface 2>/dev/null; then ip link set $iface master br0; fi; done
    host1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.14

  links:
    - endpoints: ["r1:eth1", "sw1:eth1"]
    - endpoints: ["r2:eth1", "sw1:eth2"]
    - endpoints: ["host1:eth1", "sw1:eth3"]

mgmt:
  network: custom
  ipv4-subnet: 172.20.20.0/24
```

<details>
    <summary>YAML Configuration Explained</summary>

**Router Configuration**

- `kind: linux` - This tells containerlab this is a generic Linux container
- `image: frrouting/frr:latest` - Uses the FRRouting container image for routing functionality
- `mgmt-ipv4: 172.20.20.11` - Assigns a static management IP address for SSH/API access
- `sysctl -w net.ipv4.ip_forward=1` - Enables IPv4 packet forwarding (required for routing)
- `sysctl -w net.ipv6.conf.all.forwarding=1` - Enables IPv6 forwarding
- `sleep 5` - Waits 5 seconds for FRR services to initialize
- `sed -i "s/ospfd=no/ospfd=yes/" /etc/frr/daemons` - Enables the OSPF daemon in FRR config
- `/usr/lib/frr/ospfd -d` - Starts the OSPF daemon in background mode

**Switch Configuration**

- `alpine:latest` - Lightweight Linux distribution
- `ip link add br0 type bridge` - Creates a Linux bridge named br0
- `ip link set br0 up` - Activates the bridge
- `for loop` - Adds any existing eth1, eth2, eth3 interfaces to the bridge, making sw1 act like a switch

**Links**

```bash
links:
- endpoints: ["r1:eth1", "sw1:eth1"]
- endpoints: ["r2:eth1", "sw1:eth2"]
- endpoints: ["host1:eth1", "sw1:eth3"]

Topology Visual:
r1 ---- sw1 ---- r2
         |
       host1
```

</details>

Then deployed the lab.

```bash
sudo containerlab deploy -t basic-network.clab.yml
```

:::info

Initial deployment takes 2-3 minutes as Docker pulls the required images. Subsequent deployments are much faster.

:::

I then checked the running containers.

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

Which will show something like this:

```bash
NAMES                      STATUS       IMAGE
clab-basic-network-r2      Up 8 hours   frrouting/frr:latest
clab-basic-network-sw1     Up 8 hours   alpine:latest
clab-basic-network-r1      Up 8 hours   frrouting/frr:latest
clab-basic-network-host1   Up 8 hours   alpine:latest
```

To get management IP addresses, I ran the following:

```bash
sudo containerlab inspect -t basic-network.clab.yml
```

And the result will look like something like this:

```bash
04:29:16 INFO Parsing & checking topology file=basic-network.clab.yml
╭──────────────────────────┬──────────────────────┬─────────┬────────────────╮
│           Name           │      Kind/Image      │  State  │ IPv4/6 Address │
├──────────────────────────┼──────────────────────┼─────────┼────────────────┤
│ clab-basic-network-host1 │ linux                │ running │ 172.20.20.14   │
│                          │ alpine:latest        │         │ N/A            │
├──────────────────────────┼──────────────────────┼─────────┼────────────────┤
│ clab-basic-network-r1    │ linux                │ running │ 172.20.20.11   │
│                          │ frrouting/frr:latest │         │ N/A            │
├──────────────────────────┼──────────────────────┼─────────┼────────────────┤
│ clab-basic-network-r2    │ linux                │ running │ 172.20.20.12   │
│                          │ frrouting/frr:latest │         │ N/A            │
├──────────────────────────┼──────────────────────┼─────────┼────────────────┤
│ clab-basic-network-sw1   │ linux                │ running │ 172.20.20.13   │
│                          │ alpine:latest        │         │ N/A            │
╰──────────────────────────┴──────────────────────┴─────────┴────────────────╯
```

For containerized environments, I used Docker Exec instead of SSH for better reliability.

:::note[Docker Exec]

**The Technical Process:**

```bash
docker exec -it container_name command
```

What happens internally:

1. Docker daemon receives the exec request
2. Container runtime (containerd/runc) creates a new process inside the existing container's namespace
3. Process isolation: The new process shares the container's filesystem, network, and process namespace
4. Direct execution: Command runs as if you were inside the container
5. Output returned: Results come back through Docker's API

**Docker Exec Path:**

```bash
Your Terminal → Docker Client → Docker Daemon → Container Runtime → Container Process
```

1. No network layer: Direct API communication
2. Namespace sharing: Inherits container's environment completely
3. Process spawning: Creates new process in existing container
4. Authentication: Uses Docker daemon permissions
5. Overhead: Minimal - direct syscalls

**SSH Path:**

```bash
Your Terminal → SSH Client → Network → Container SSH Daemon → Container Process
```

1. Network protocol: TCP connection with encryption
2. Authentication layer: Username/password or keys
3. SSH daemon: Requires sshd running inside container
4. Session management: Full SSH session with environment setup
5. Overhead: Network handshake, encryption, session management

I'll be using a hybrid approach.

- Ansible server uses SSH to reach Docker host
- Then uses Docker exec to reach containers

```bash title="Example Command"
ssh containerlab-server "docker exec router1 vtysh -c 'show version'"
```

:::

I tested direct container access to the routers on the ContainerLab host.

```bash
docker exec -it clab-basic-network-r1 vtysh -c "show version"
docker exec -it clab-basic-network-r2 vtysh -c "show version"
```

Then tested the Linux host1 and sw1.

```bash
docker exec -it clab-basic-network-r1 /bin/bash
docker exec -it clab-basic-network-host1 /bin/sh
```

## Ansible Installation

As always, I updated the system.

```bash
sudo apt update && sudo apt upgrade -y
```

I will also install Docker, because I'll be using Docker Exec instead of SSH to communicate with the Containerlab devices.

```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

Then installed Python 3 and pip.

```bash
sudo apt install -y python3 python3-pip python3-venv
```

### Python Virtual Environment

Next I'll be creating a Python virtual environment for Ansible.

:::note[What is a Python Virtual Environment?]

A virtual environment is an isolated Python workspace that keeps your project dependencies separate from your system's Python installation and other projects.

:::

First, I created the directory.

```bash
mkdir -p ~/ansible-env
cd ~/ansible-env
```

Then created the virtual environment and activated it.

```bash
python3 -m venv ansible-venv
source ansible-venv/bin/activate
```

:::info[Command Explanation]

- `python3` - Use Python 3 interpreter
- `-m venv` - Runs the `venv` module (virtual environment creator)
- `ansible-venv` - Creates a directory with this name for the virtual environment
- `source` - Runs a script in the current shell (not in a new process)
- `ansible-venv/bin/activate` - The activation script

:::

:::note[Why Use virtual Environments?]

**Without Virtual Environment:**

```bash
# System-wide installation
sudo pip install ansible==5.0
sudo pip install some-other-tool  # Might need ansible==6.0
# Conflict! One breaks the other
```

**With Virtual Environment:**

```bash
# Project 1 environment
python3 -m venv project1-env
source project1-env/bin/activate
pip install ansible==5.0  # Isolated to this project

# Project 2 environment
python3 -m venv project2-env
source project2-env/bin/activate
pip install ansible==6.0  # Different version, no conflict
```

**Key Benefits**

1. **Isolation**: Each project has its own package versions
2. **No `sudo` needed**: Install packages without admin rights
3. **Reproducible**: Can recreate exact environment on other machines
4. **Clean**: No conflicts between different projects
5. **Portable**: Can delete entire environment easily

:::

I then installed Ansible and required collections.

```bash
pip install ansible
pip install netaddr
```

I also installed Ansible Galaxy collections for container automation.

```bash
ansible-galaxy collection install community.docker
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
```

Then verified the installation.

```bash
ansible --version
```

### Ansible Project Structure

Next, I created the Ansible project structure.

```bash
mkdir -p ~/network-automation/{playbooks,inventory,group_vars,host_vars,roles,files,templates}
cd ~/network-automation
```

:::info[Command Explanation]

- `playbooks/` - Ansible playbooks (automation scripts)
- `inventory/` - Host inventory files
- `group_vars/` - Variables for groups of hosts
- `host_vars/` - Variables for individual hosts
- `roles/` - Reusable Ansible roles
- `files/` - Static files to copy to hosts
- `templates/` - Jinja2 templates for dynamic configs

:::

Next, I created the ansible.cfg file.

```bash title="ansible.cfg"
[defaults]
inventory = inventory/hosts.yml
host_key_checking = False
timeout = 60
gathering = explicit
retry_files_enabled = False
command_warnings = False

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

:::info[Command Explanation]

- `inventory = inventory/hosts.yml` - Tells Ansible where to find your inventory file.
- `host_key_checking = False` - Disables SSH host key verification (only used for testing)
- `timeout = 60` - Sets connection timeout to 60 seconds
- `gathering = explicit` - Disables automatic fact gathering (by default, Ansible collects ssystem information from each host before running tasks)
- `retry_files_enabled = false` - Disables creation of `.retry` files (When playbooks fail, Ansible normally creates files listing failed hosts)
- `command_warnings = False` - Suppresses warnings about using certain Ansible modules
- `enable_plugins = ...` - Specifies which inventory plugins Ansible can use

:::

Then, I created the inventory file.

```yaml title="inventory/hosts.yml"
---
all:
  children:
    containerlab_devices:
      children:
        routers:
          hosts:
            r1:
              ansible_host: clab-basic-network-r1
              ansible_connection: community.docker.docker
            r2:
              ansible_host: clab-basic-network-r2
              ansible_connection: community.docker.docker
        hosts:
          hosts:
            host1:
              ansible_host: clab-basic-network-host1
              ansible_connection: community.docker.docker
            sw1:
              ansible_host: clab-basic-network-sw1
              ansible_connection: community.docker.docker
      vars:
        ansible_user: root
        ansible_docker_extra_args: ""
```

<details>
    <summary>Inventory File Explained</summary>
**Hierarchy:**

```bash
all
└── containerlab_devices
    ├── routers (r1, r2)
    └── hosts (host1, sw1)
```

**Key Parameters:**

- `ansible_host` - The Docker container name
- `ansible_connection: community.docker.docker` - Uses Docker exec instead of SSH
- `ansible_user: root` - Connects as root user inside containers
- `ansible_docker_extra_args` - Additional Docker arguments (empty for now)

**This structure allows you to:**

- Target all devices: `--limit containerlab_devices`
- Target only routers: `--limit routers`
- Target specific device: `--limit r1`

</details>

### Setting Up Passwordless SSH

First, I needed to see if my Ansible server can reach my Containerlab server.

```bash
ping -c 3 10.33.99.12
```

Next, I setup passwordless SSH, for testing.

So, I generated an SSH key.

```bash
ssh-keygen -t rsa -b 4096
```

:::info

Press Enter for all prompts to use default settings

:::

Then copied the key to my ContainerLab server.

```bash
ssh-copy-id nesto@10.33.99.12
```

Then tested.

```bash
ssh nesto@10.33.99.12 'docker ps'
```

:::note[Example Output]

```bash
nesto@ansible:~$ ssh nesto@10.33.99.12 'docker ps'
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS        PORTS     NAMES
6de509474d32   frrouting/frr:latest   "/sbin/tini -- /usr/…"   27 hours ago   Up 27 hours             clab-basic-network-r2
83ab586ed547   alpine:latest          "/bin/sh"                27 hours ago   Up 27 hours             clab-basic-network-sw1
5bf49eca32d9   frrouting/frr:latest   "/sbin/tini -- /usr/…"   27 hours ago   Up 27 hours             clab-basic-network-r1
872304df5388   alpine:latest          "/bin/sh"                27 hours ago   Up 27 hours             clab-basic-network-host1
```

:::

## Creating Ansible Playbooks

### Test Connectivity

Next, I created a basic playbook for testing.

```yaml title="playbooks/test-connectivity.yml"
---
- name: Test connectivity to FRRouting devices
  hosts: routers
  gather_facts: no
  tasks:
    - name: Check if FRR is running
      ansible.builtin.command: vtysh -c "show version"
      register: frr_version

    - name: Display FRR version
      debug:
        msg: "{{ frr_version.stdout_lines }}"

- name: Test connectivity to hosts
  hosts: hosts
  gather_facts: yes
  tasks:
    - name: Display host information
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }}
          IP: {{ ansible_default_ipv4.address }}
```

<details>
  <summary>Playbook Explained</summary>

**Play 1: FRRouting Devices**

- `---` - YAML document start marker (required in Ansible)
- `gather_facts: no` - Disables fact gathering. Faster startup since it will skip the automatic system discovery phase
- `ansible.builtin.command:` - Runs shell commands
- `vtysh -c "show version"` - is the FRR CLI command
- `debug` - Prints information to the console (like `echo` or `print`)
- `msg: "{{ frr_version.stdout_lines }}` - The output from the previous command, split into lines

**Play 2: Linux Hosts**

- `ansible.builtin.raw` - Executes raw commands (works on Alpine without Python)
- Uses shell commands to gather hostname and IP information
- `changed_when: false` - Read-only operation, no changes made

**Example Output**

```bash
PLAY [Test connectivity to FRRouting devices] *********************************

TASK [Check if FRR is running] ************************************************
ok: [r1]
ok: [r2]

TASK [Display FRR version] ****************************************************
ok: [r1] => {
    "msg": [
        "FRRouting 8.4_git (r1) on Linux(5.15.0-156-generic).",
        "Copyright 1996-2005 Kunihiro Ishiguro, et al."
    ]
}
ok: [r2] => {
    "msg": [
        "FRRouting 8.4_git (r2) on Linux(5.15.0-156-generic).",
        "Copyright 1996-2005 Kunihiro Ishiguro, et al."
    ]
}

PLAY [Test connectivity to hosts] *********************************************

TASK [Gathering Facts] ********************************************************
ok: [host1]
ok: [sw1]

TASK [Display host information] **********************************************
ok: [host1] => {
    "msg": "Hostname: clab-basic-network-host1\nOS: Alpine\nIP: 172.20.20.14\n"
}
ok: [sw1] => {
    "msg": "Hostname: clab-basic-network-sw1\nOS: Alpine\nIP: 172.20.20.13\n"
}
```

</details>

Now I ran the playbook to verify everything is working.

```bash
cd ~/network-automation
source ~/ansible-env/ansible-venv/bin/activate
export DOCKER_HOST=ssh://nesto@10.33.99.12
ansible-playbook playbooks/test-connectivity.yml
```

:::warning

`export DOCKER_HOST=ssh://nesto@10.33.99.12` tells Ansible to connect to Docker on the remote ContainerLab server via SSH. You'll need to set this for every new terminal session, or add it to your `~/.bashrc` file.

:::

:::note[Example Output]

```bash
(ansible-venv) nesto@ansible:~/network-automation$ ansible-playbook playbooks/test-connectivity.yml

PLAY [Test connectivity to FRRouting devices] ****************************************************************************************************************************************************************

TASK [Check if FRR is running] *******************************************************************************************************************************************************************************
[WARNING]: Platform linux on host r2 is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [r2]
[WARNING]: Platform linux on host r1 is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [r1]

TASK [Display FRR version] ***********************************************************************************************************************************************************************************
ok: [r1] => {
    "msg": [
        "FRRouting 8.4_git (r1) on Linux(5.15.0-156-generic).",
        "Copyright 1996-2005 Kunihiro Ishiguro, et al.",
        "configured with:",
        "    '--prefix=/usr' '--sbindir=/usr/lib/frr' '--sysconfdir=/etc/frr' '--libdir=/usr/lib' '--localstatedir=/var/run/frr' '--enable-rpki' '--enable-vtysh' '--enable-multipath=64' '--enable-vty-group=frrvty' '--enable-user=frr' '--enable-group=frr' '--enable-pcre2posix' 'CC=gcc' 'CXX=g++'"
    ]
}
ok: [r2] => {
    "msg": [
        "FRRouting 8.4_git (r2) on Linux(5.15.0-156-generic).",
        "Copyright 1996-2005 Kunihiro Ishiguro, et al.",
        "configured with:",
        "    '--prefix=/usr' '--sbindir=/usr/lib/frr' '--sysconfdir=/etc/frr' '--libdir=/usr/lib' '--localstatedir=/var/run/frr' '--enable-rpki' '--enable-vtysh' '--enable-multipath=64' '--enable-vty-group=frrvty' '--enable-user=frr' '--enable-group=frr' '--enable-pcre2posix' 'CC=gcc' 'CXX=g++'"
    ]
}

PLAY [Test connectivity to hosts] ****************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************************************
fatal: [sw1]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"}, "failed": true, "module_stderr": "/bin/sh: /usr/bin/python3: not found\n", "module_stdout": "", "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error", "rc": 127, "warnings": ["No python interpreters found for host sw1 (tried ['python3.12', 'python3.11', 'python3.10', 'python3.9', 'python3.8', 'python3.7', '/usr/bin/python3', 'python3'])"]}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}
fatal: [host1]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"}, "failed": true, "module_stderr": "/bin/sh: /usr/bin/python3: not found\n", "module_stdout": "", "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error", "rc": 127, "warnings": ["No python interpreters found for host host1 (tried ['python3.12', 'python3.11', 'python3.10', 'python3.9', 'python3.8', 'python3.7', '/usr/bin/python3', 'python3'])"]}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}

PLAY RECAP ***************************************************************************************************************************************************************************************************
host1                      : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
r1                         : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
r2                         : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
sw1                        : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

### Network Configuration

Now I created another playbook to make some basic configuration changes on the routers.

```yaml title="playbooks/basic-config.yml"
---
- name: Configure FRRouting devices
  hosts: routers
  gather_facts: no
  tasks:
    - name: Configure OSPF on routers (correct FRR syntax)
      ansible.builtin.shell: |
        vtysh -c 'configure terminal' \
              -c 'router ospf' \
              -c 'network 172.20.20.0/24 area 0' \
              -c 'exit' \
              -c 'exit' \
              -c 'write'

    - name: Configure interface descriptions in FRR
      ansible.builtin.shell: |
        vtysh -c 'configure terminal' \
              -c 'interface eth1' \
              -c 'description Connection to switch' \
              -c 'exit' \
              -c 'exit' \
              -c 'write'

    - name: Configure IP addresses using Linux commands
      ansible.builtin.shell: |
        ip addr add {{ router_ip }}/24 dev eth1 || true
        ip link set eth1 up
      vars:
        router_ip: "{{ '10.1.1.1' if 'r1' in ansible_host else '10.1.2.1' }}"

- name: Configure host networking (fixed for Alpine)
  hosts: hosts
  gather_facts: no
  tasks:
    - name: Set up basic networking on hosts (using raw for Alpine)
      ansible.builtin.raw: |
        ip addr add {{ host_ip }}/24 dev eth1 2>/dev/null || echo "IP may already exist"
        ip link set eth1 up 2>/dev/null || echo "Interface may already be up"
        echo "Host {{ ansible_host }} configured with {{ host_ip }}"
      vars:
        host_ip: "{{ '10.1.100.1' if 'host1' in ansible_host else '10.1.100.2' }}"
```

<details>
    <summary>YAML File Explained</summary>

1. Play 1: Configures FRRouting Routers

**OSPF Configuraition**

- `-c 'configure terminal'` - Enter config mode
- `-c 'router ospf'` - Enter OSPF router configuration
- `-c 'network 172.20.20.0/24 area 0'` - Advertise management network in OSPF area 0
- `-c 'exit'` - Exit OSPF config mode
- `-c 'exit'` - Exit global config mode
- `-c 'write'` - Save configuration

**Interface Configuration**

- `-c 'interface eth1'` - Configure interface eth1
- `-c 'description Connection to switch'` - Add interface description

**IP Address Configuration**

- Uses Linux `ip` command (not FRR)
- `|| true` prevents errors if IP already exists
- `vars:` section assigns different IPs to each router
- r1 gets 10.1.1.1, r2 gets 10.1.2.1

1. Play 2: Configures Linux Hosts

- Uses `raw` module (Alpine doesn't have Python)
- Configures IP addresses on eth1 interface
- Brings interface up
- host1 gets 10.1.100.1, sw1 gets 10.1.100.2

</details>

I then ran the playbook.

```bash
cd ~/network-automation
source ~/ansible-env/ansible-venv/bin/activate
export DOCKER_HOST=ssh://nesto@10.33.99.12
ansible-playbook playbooks/basic-config.yml
```

:::note[Sample Output]

```bash
(ansible-venv) nesto@ansible:~/network-automation$ ansible-playbook playbooks/basic-config.yml

PLAY [Configure FRRouting devices] ***************************************************************************************************************************************************************************

TASK [Configure OSPF on routers (correct FRR syntax)] ********************************************************************************************************************************************************
[WARNING]: Platform linux on host r1 is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [r1]
[WARNING]: Platform linux on host r2 is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [r2]

TASK [Configure interface descriptions in FRR] ***************************************************************************************************************************************************************
changed: [r1]
changed: [r2]

TASK [Configure IP addresses using Linux commands] ***********************************************************************************************************************************************************
changed: [r1]
changed: [r2]

PLAY [Configure host networking (fixed for Alpine)] **********************************************************************************************************************************************************

TASK [Set up basic networking on hosts (using raw for Alpine)] ***********************************************************************************************************************************************
changed: [host1]
changed: [sw1]

PLAY RECAP ***************************************************************************************************************************************************************************************************
host1                      : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
r1                         : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
r2                         : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
sw1                        : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

:::

### Verify Configuration

After running the configuration playbook, I verified everything is working correctly.

I checked OSPF neighbors.

```json
ssh nesto@10.33.99.12 "docker exec clab-basic-network-r1 vtysh -c 'show ip ospf neighbor'"
```

I then also checked IP addresses.

```json
ssh nesto@10.33.99.12 "docker exec clab-basic-network-r1 ip addr show eth1"
ssh nesto@10.33.99.12 "docker exec clab-basic-network-r2 ip addr show eth1"
```

Then tested connectivity between routers.

```json
ssh nesto@10.33.99.12 "docker exec clab-basic-network-r1 ping -c 3 10.1.2.1"
```

:::note[Idempotency]

Idempotency means you can run the same playbook multiple times without causing problems.

:::

## Managing Lab Lifecycle

### Stopping and Starting

To stop and remove all lab containers, run the following on the ContainerLab server:

```bash
sudo containerlab destroy -t basic-network.clab.yml
```

Or, if you're in the directory with only one .clab.yml file, run:

```bash
sudo containerlab destroy
```

To restart the lab:

```bash
sudo containerlab deploy -t basic-network.clab.yml
```

::: Checking Lab Status

To check which labs are running, run:

```bash
sudo containerlab inspect --all
```

To verify containers are stopped, run:

```bash
docker ps
```

### Cleanup Operations

To cleanup everything (containers + leftover files) run:

```bash
sudo containerlab destroy -t basic-network.clab.yml --cleanup
```

To force destory if normal destory fails, run:

```bash
sudo containerlab destroy -t basic-network.clab.yml --force
```

To destroy by lab name instead of file, run:

```bash
sudo containerlab destroy --name basic-network
```

To remove all stopped containers, run:

```bash
docker container prune
```

To remove unused Docker images, run:

```bash
docker image prune -a
```

## Useful Commands

### Docker Exec

To connect to a router directly using Docker Exec:

```bash
docker exec -it clab-basic-network-r1 vtysh
```

To run multiple commands:

```bash
docker exec clab-basic-network-r1 vtysh \
  -c "configure terminal" \
  -c "router ospf" \
  -c "area 0 stub" \
  -c "exit" \
  -c "exit" \
  -c "write"
```

To access container shell for system level changes:

```bash
docker exec -it clab-basic-network-r1 /bin/bash
```

To view container resource usage:

```bash
docker stats clab-basic-network-r1
```

### SSH

To SSH to the router:

```bash
ssh root@172.20.20.11
```

From the Ansible server using the Docker host as a jump:

```bash
ssh -J nesto@10.33.99.12 root@172.20.20.11
```

To copy files to container vis jump host:

```bash
scp -J nesto@10.33.99.12 myfile.conf root@172.20.20.11:/etc/frr/
```

### Ansible Ad-Hoc

To configure single interface:

```bash
ansible routers -m shell -a "vtysh -c 'configure terminal' \
  -c 'interface eth1' \
  -c 'ip ospf cost 100' \
  -c 'exit' \
  -c 'exit' \
  -c 'write'" --limit r1
```

For an interactive session:

```bash
ansible routers -m shell -a "vtysh" --limit r1
```

### FRRouting Verification

To check OSPF status:

```bash
bashdocker exec clab-basic-network-r1 vtysh -c "show ip ospf"
docker exec clab-basic-network-r1 vtysh -c "show ip ospf neighbor"
docker exec clab-basic-network-r1 vtysh -c "show ip ospf database"
```

To view routing table:

```bash
bashdocker exec clab-basic-network-r1 vtysh -c "show ip route"
docker exec clab-basic-network-r1 vtysh -c "show ip route ospf"
```

To view Interface status:

```bash
bashdocker exec clab-basic-network-r1 vtysh -c "show interface"
docker exec clab-basic-network-r1 vtysh -c "show interface eth1"
```

To view configuration:

```bash
bashdocker exec clab-basic-network-r1 vtysh -c "show running-config"
docker exec clab-basic-network-r1 vtysh -c "show startup-config"
```