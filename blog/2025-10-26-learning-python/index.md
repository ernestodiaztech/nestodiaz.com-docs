---
slug: learning-python
title: Python Lab Setup
authors: [nesto]
tags: [python, cisco, linux]
---

This weekend I setup a lab to learn Python, and Ansible. I used ContainerLab so I can easily deploy and destroy labs.

A quick overview of what needed to be done. This is not a complete setup guide.

I created the server for ContainerLab on my Proxmox server. I used Ubuntu 22.04 as the OS, and gave the VM 64GB of RAM, 100GB of storage, and 16 CPU cores.

This is my current ContainerLab YML file to start learning the basics of Python and Ansible.

```bash title="automation-lab.clab.yml"
name: automation-lab

topology:
  nodes:
    router1:
      kind: cisco_c8000v
      image: vrnetlab/cisco_c8000v:17.12.01
      mgmt-ipv4: 172.20.20.11
      
    router2:
      kind: cisco_c8000v
      image: vrnetlab/cisco_c8000v:17.12.01
      mgmt-ipv4: 172.20.20.12

    switch1:
      kind: cisco_cat9kv
      image: vrnetlab/cisco_cat9kv:17.12.01
      mgmt-ipv4: 172.20.20.21
      
    switch2:
      kind: cisco_cat9kv
      image: vrnetlab/cisco_cat9kv:17.12.01
      mgmt-ipv4: 172.20.20.22
      
    switch3:
      kind: cisco_cat9kv
      image: vrnetlab/cisco_cat9kv:17.12.01
      mgmt-ipv4: 172.20.20.23

    ubuntu1:
      kind: linux
      image: ubuntu:22.04
      mgmt-ipv4: 172.20.20.31
      exec:
        - apt update
        - apt install -y iproute2 iputils-ping net-tools curl wget vim
        - ip addr add 10.10.1.10/24 dev eth1 || true
        - ip link set eth1 up
        - ip route add default via 10.10.1.1 || true
        
    ubuntu2:
      kind: linux
      image: ubuntu:22.04
      mgmt-ipv4: 172.20.20.32
      exec:
        - apt update
        - apt install -y iproute2 iputils-ping net-tools curl wget vim
        - ip addr add 10.10.2.10/24 dev eth1 || true
        - ip link set eth1 up
        - ip route add default via 10.10.2.1 || true
        
    ubuntu3:
      kind: linux
      image: ubuntu:22.04
      mgmt-ipv4: 172.20.20.33
      exec:
        - apt update
        - apt install -y iproute2 iputils-ping net-tools curl wget vim
        - ip addr add 10.10.3.10/24 dev eth1 || true
        - ip link set eth1 up
        - ip route add default via 10.10.3.1 || true

  links:
    # Router to Router connection
    - endpoints: ["router1:eth1", "router2:eth1"]
    
    # Router to Switch connections
    - endpoints: ["router1:eth2", "switch1:eth1"]
    - endpoints: ["router2:eth2", "switch2:eth1"]
    
    # Switch to Switch connections
    - endpoints: ["switch1:eth2", "switch2:eth2"]
    - endpoints: ["switch1:eth3", "switch3:eth1"]
    
    # Switch to Host connections
    - endpoints: ["switch3:eth2", "ubuntu1:eth1"]
    - endpoints: ["switch3:eth3", "ubuntu2:eth1"]
    - endpoints: ["switch2:eth3", "ubuntu3:eth1"]

mgmt:
  network: custom
  ipv4-subnet: 172.20.20.0/24
  ```

To start the lab I ran:

```bash
containerlab deploy -t /containerlab/topologies/automation-lab.clab.yml
```

Once it's finished deploying I get the following:

```bash
╭─────────────────────────────┬────────────────────────────────┬────────────────────┬────────────────╮
│             Name            │           Kind/Image           │        State       │ IPv4/6 Address │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-c8kv-router1 │ cisco_c8000v                   │ running            │ 172.20.20.11   │
│                             │ vrnetlab/cisco_c8000v:17.06.02 │ (health: starting) │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-c8kv-router2 │ cisco_c8000v                   │ running            │ 172.20.20.12   │
│                             │ vrnetlab/cisco_c8000v:17.06.02 │ (health: starting) │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-c9kv-switch1 │ cisco_cat9kv                   │ running            │ 172.20.20.21   │
│                             │ vrnetlab/cisco_cat9kv:17.15.01 │ (health: starting) │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-c9kv-switch2 │ cisco_cat9kv                   │ running            │ 172.20.20.22   │
│                             │ vrnetlab/cisco_cat9kv:17.15.01 │ (health: starting) │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-c9kv-switch3 │ cisco_cat9kv                   │ running            │ 172.20.20.23   │
│                             │ vrnetlab/cisco_cat9kv:17.15.01 │ (health: starting) │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-ubuntu1      │ linux                          │ running            │ 172.20.20.31   │
│                             │ ubuntu:22.04                   │                    │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-ubuntu2      │ linux                          │ running            │ 172.20.20.32   │
│                             │ ubuntu:22.04                   │                    │ N/A            │
├─────────────────────────────┼────────────────────────────────┼────────────────────┼────────────────┤
│ clab-cisco-lab-ubuntu3      │ linux                          │ running            │ 172.20.20.33   │
│                             │ ubuntu:22.04                   │                    │ N/A            │
╰─────────────────────────────┴────────────────────────────────┴────────────────────┴────────────────╯
```

Then after a few minutes I checked if everything is up and running with `docker ps`.

```bash
CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS                   PORTS                                                 NAMES
26ef3fe4da3f   vrnetlab/cisco_cat9kv:17.15.01   "/launch.py --userna…"   7 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-cisco-lab-c9kv-switch1
199fc99c9412   vrnetlab/cisco_cat9kv:17.15.01   "/launch.py --userna…"   7 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-cisco-lab-c9kv-switch3
008f1dc9cc7f   vrnetlab/cisco_cat9kv:17.15.01   "/launch.py --userna…"   7 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-cisco-lab-c9kv-switch2
4c78a60cba3f   vrnetlab/cisco_c8000v:17.06.02   "/launch.py --userna…"   7 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-cisco-lab-c8kv-router2
7b00c965c735   vrnetlab/cisco_c8000v:17.06.02   "/launch.py --userna…"   7 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-cisco-lab-c8kv-router1
3d0db5ba6513   ubuntu:22.04                     "/bin/bash"              7 minutes ago   Up 7 minutes                                                                   clab-cisco-lab-ubuntu3
a2be32994a9f   ubuntu:22.04                     "/bin/bash"              7 minutes ago   Up 7 minutes                                                                   clab-cisco-lab-ubuntu2
0ab971ac58c7   ubuntu:22.04                     "/bin/bash"              7 minutes ago   Up 7 minutes                                                                   clab-cisco-lab-ubuntu1
```

I checked if I'm able to ping each device.

```bash
nesto@clab:~/containerlab/topologies$ ping 172.20.20.33
PING 172.20.20.33 (172.20.20.33) 56(84) bytes of data.
64 bytes from 172.20.20.33: icmp_seq=1 ttl=64 time=0.085 ms
64 bytes from 172.20.20.33: icmp_seq=2 ttl=64 time=0.080 ms
64 bytes from 172.20.20.33: icmp_seq=3 ttl=64 time=0.085 ms
64 bytes from 172.20.20.33: icmp_seq=4 ttl=64 time=0.076 ms
64 bytes from 172.20.20.33: icmp_seq=5 ttl=64 time=0.073 ms
64 bytes from 172.20.20.33: icmp_seq=6 ttl=64 time=0.079 ms
64 bytes from 172.20.20.33: icmp_seq=7 ttl=64 time=0.089 ms
```

On the Ubuntu server I had to install Python3, Pip, Paramiko and Netmiko.

```bash
sudo apt install python3
sudo apt install python3-pip -y
pip install paramiko
pip install netmiko
```

To connect to the routers and switches I will run:

```bash
docker exec -it clab-cisco-lab-c8kv-router1 telnet 127.0.0.1 5000
```

To exit out of the session:

```bash
Ctrl+]
quit
```

To connect to the Ubuntu servers I will run:

```bash
docker exec -it clab-cisco-lab-ubuntu1 bash
```

