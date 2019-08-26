# Ansible Docker etcd Cluster

[![Work with Ansible](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg)](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) 

An Ansible playbook setups an etcd cluster with Docker and creates an overlay network.

This is an example for learning Docker overlay networks and service discovery with etcd. The newer approach is using Docker swarm mode like [ansible-docker-swarm](https://github.com/yungshun317/ansible-docker-swarm). Right now Docker swarm mode doesn't need an external key-value store. 

## Usage

You have 3 nodes available.

Run `ansible-playbook`.
```sh
$ ansible-playbook etcd-cluster.yml
```

## Overview

In all nodes:

1. Ensure Python 2.7 is definitely present for running Ansible.

2. Install required Docker packages. Then add user to the docker group and reboot. For more details, please refer to [ansible-docker-postgres-remote](https://github.com/yungshun317/ansible-docker-postgres-remote).

3. Restart Docker engine with cluster configuration by `nohup /usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://{{ ansible_default_ipv4.address }}:2379 --cluster-advertise={{ ansible_default_ipv4.address }}:2375 >/dev/null 2>&1 &`. Compare this configuration with [ansible-docker-swarm](https://github.com/yungshun317/ansible-docker-swarm).

4. Create a directory, `/home/ccma/etcd`, for mounting host volume.

5. Stop and remove the `etcd` containers if they already exists with the same approach described in [ansible-docker-postgres-remote](https://github.com/yungshun317/ansible-docker-postgres-remote).

6. Run a 3 node etcd cluster with `docker_container`.
```yaml
- name: Run a 3 node etcd cluster
  docker_container:
    image: gcr.io/etcd-development/etcd:latest
    name: etcd
    volumes:
      - /home/ccma/etcd:/etcd-data
    ports:
      - "2379:2379"
      - "2380:2380"
    command: [
        "/usr/local/bin/etcd",
        "--data-dir=/etcd-data",
        "--name {{ hostvars[inventory_hostname]['etcd_name'] }}",
        "--initial-advertise-peer-urls http://{{ ansible_default_ipv4.address }}:2380",
        "--listen-peer-urls http://0.0.0.0:2380",
        "--advertise-client-urls http://{{ ansible_default_ipv4.address }}:2379",
        "--listen-client-urls http://0.0.0.0:2379",
        "--initial-cluster {{ cluster }}",
        "--initial-cluster-state {{ cluster_state }}",
        "--initial-cluster-token {{ token }}"
    ]
  vars:
      cluster: "etcd1=http://10.236.1.1:2380,etcd2=http://10.236.1.2:2380,etcd3=http://10.236.1.3:2380"
      cluster_state: "new"
      token: "etcd-cluster"
  register: run_etcd_cluster
```
Remember to set `etcd_name` in your `host_vars`.

In one node:

1. Delete the old `network_etcd` overlay network and re-create one.

```yaml
- name: Delete a network, disconnecting all containers
  docker_network:
    name: network_etcd
    state: absent
    force: yes

- name: Create a network
  become: yes
  shell: "docker network create -d overlay network_etcd"
```

2. Check the network list.

```yaml
- name: Check the network list 
  shell: "docker network ls"
   register: docker_network_ls

- name: Debug docker_network_ls
  debug:
    var: docker_network_ls 
```

## Testing

After running a Docker etcd cluster, test by
```sh
$ curl -L http://10.236.1.1:2379/version
$ curl -L http://10.236.1.1:2379/health

$ docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl member list" 
c5b0059ac3a25bd, started, etcd3, http://10.236.1.3:2380, http://10.236.1.3:2379
92da022a63751f46, started, etcd1, http://10.236.1.1:2380, http://10.236.1.1:2379
f4d580369ca924f6, started, etcd2, http://10.236.1.2:2380, http://10.236.1.2:2379

$ docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl put service_name service_address"
OK
$ docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl get service_name"
service_name
service_address
$ docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl del service_name"
1
```

In the first node (`etcd1`).
```sh
$ docker run -d --name test1 --net network_etcd busybox sh -c "while true; do sleep 3600; done"
$ docker ps
$ docker exec test1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:0A:00:00:02  
          inet addr:10.0.0.2  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::42:aff:fe00:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1038 (1.0 KiB)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:AC:12:00:02  
          inet addr:172.18.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe12:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1038 (1.0 KiB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

In the another node (`etcd2`).
```sh
$ docker run -d --name test1 --net network_etcd busybox sh -c "while true; do sleep 3600; done"
docker: Error response from daemon: Conflict. The container name "/test1" is already in use by container 0bf5b1e94dd2b03af4ecac587bc28061189a0d8739731a4198f8937535b06c7e. You have to remove (or rename) that container to be able to reuse that name..

$ docker run -d --name test2 --net network_etcd busybox sh -c "while true; do sleep 3600; done"

$ docker network inspect network_etcd
[
    {
        "Name": "network_etcd",
        "Id": "b171099b961935c08aeef34f17c8611e372fa6bcc76dbedf7cf6bf4c86c3daf8",
        "Created": "2018-11-01T13:46:43.407153744+08:00",
        "Scope": "global",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.0.0.0/24",
                    "Gateway": "10.0.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "ep-09d9b6e5bcb194e668174a838509dc407de092759867e8621058e627c7e820a7": {
                "Name": "test2",
                "EndpointID": "09d9b6e5bcb194e668174a838509dc407de092759867e8621058e627c7e820a7",
                "MacAddress": "02:42:0a:00:00:03",
                "IPv4Address": "10.0.0.3/24",
                "IPv6Address": ""
            },
            "ep-695294e2ccf486d38b02e1a30952835344d9b09705e046d5d6517b596837eb5b": {
                "Name": "test1",
                "EndpointID": "695294e2ccf486d38b02e1a30952835344d9b09705e046d5d6517b596837eb5b",
                "MacAddress": "02:42:0a:00:00:02",
                "IPv4Address": "10.0.0.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## Tech
This project uses:
* [Ansible](https://www.ansible.com/) - Ansible is open source software that automates software provisioning, configuration management, and application deployment.
* [Docker](https://github.com/docker/docker-ce) - a computer program that performs operating-system-level virtualization, also known as containerization.
* [etcd](https://etcd.io/) - a distributed, reliable key-value store for the most critical data of a distributed system.

## Todos
 - Dive deeply in container networking with Kubernetes and Istio.
 - Try to write an Ansible Role.

## License
[Ansible Docker etcd Cluster](https://github.com/yungshun317/ansible-docker-etcd-cluster) is released under the [MIT License](https://opensource.org/licenses/MIT) by [yungshun317](https://github.com/yungshun317).
