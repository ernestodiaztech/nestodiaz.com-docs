---
sidebar_position: 1
---

# 1. Installing Container Lab

## Network Overview

| Server       | IP          | Subnet        |
| ------------ | ----------- | ------------- |
| ContainerLab | 10.33.99.12 | 255.255.255.0 |
| Ansible      | 10.33.99.13 | 255.255.255.0 |

## ContainerLab Installation

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

Then ran instead of logging out and logging back in.

```bash
newgrp docker
```

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

Next, I created a simple topology for testing.

I made a directory for my topologies.

```bash
mkdir -p ~/containerlab/topologies
cd ~/containerlab/topologies
```

Then made a basic network topology.

```bash title="basic-network.clab.yml"
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

Then deployed the lab.

```bash
sudo containerlab deploy -t basic-network.clab.yml
```

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

Then created the Ansible project structure.

```bash
mkdir -p ~/network-automation/{playbooks,inventory,group_vars,host_vars,roles,files,templates}
cd ~/network-automation
```

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

```bash title="inventory/hosts.yml"
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

I now needed to see if my Ansible server can reach my Containerlab server.

```bash
ping -c 3 10.33.99.12
```

Next, I setup passwordless access, for testing.

So, I generated an SSH key.

```bash
ssh-keygen -t rsa -b 4096
```

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

## Playbooks

Next, I created a basic playbook for testing.

```bash title="playbooks/test-connectivity.yml"
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
  <summary>Playbook File Explained</summary>

- `---` - YAML document start marker (required in Ansible)
- `gather_facts: no` - Disables fact gathering. Faster startup since it will skip the automatic system discovery phase
- `ansible.builtin.command:` - Runs shell commands
- `vtysh -c "show version"` - is the FRR CLI command
- `debug` - Prints information to the console (like `echo` or `print`)
- `msg: "{{ frr_version.stdout_lines }}` - The output from the previous command, split into lines

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

:::

Now I created another playbook to make some basic configuration changes on the routers.

```bash title="playbooks/basic-config.yml"
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

1. Play 1: Configures FRRouting routers (OSPF routing protocol)
2. Play 2: Configures basic networking on host containers

- `-c 'configure terminal'` - Enter config mode
- `-c 'router ospf'` - Enter OSPF router configuration
- `-c 'network 172.20.20.0/24 area 0'` - Advertise management network in OSPF area 0
- `-c 'exit'` - Exit OSPF config mode
- `-c 'exit'` - Exit global config mode
- `-c 'write'` - Save configuration
- `-c 'interface eth1'` - Configure interface eth1
- `-c 'description Connection to switch'` - Add interface description
- `-c 'ip address 10.1.X.1/24'` - Set IP address (X varies per router)
- `-c 'exit'` - Exit interface config
- `-c 'exit'` - Exit global config
- `-c 'write'` - Save configuration

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

## Managing Lab Lifecycle

To stop and remove all lab containers, run the following on the ContainerLab server:

```bash
sudo containerlab destroy -t basic-network.clab.yml
```

Or, if you're in the directory with only one .clab.yml file, run:

```bash
sudo containerlab destroy
```

To check which labs are running, run:

```bash
sudo containerlab inspect --all
```

To very containers are stopped, run:

```bash
docker ps
```

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
