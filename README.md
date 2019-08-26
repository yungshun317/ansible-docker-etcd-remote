# Ansible Docker etcd Cluster

[![Work with Ansible](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg)](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) 

An Ansible playbook setups an etcd cluster with Docker.

This is an example for learning Docker overlay networks, but it's not practical to modify global Docker configurations for only used by an etcd cluster. The another approach is [ansible-docker-swarm-spark](https://github.com/yungshun317/ansible-docker-swarm-spark), which deploys the cluster using `docker-compose` in swarm mode. But both are not suitable for production. Just go for Kubernetes.

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

## Tech
This project uses:
* [Ansible](https://www.ansible.com/) - Ansible is open source software that automates software provisioning, configuration management, and application deployment.
* [Docker](https://github.com/docker/docker-ce) - a computer program that performs operating-system-level virtualization, also known as containerization.
* [etcd](https://etcd.io/) - a distributed, reliable key-value store for the most critical data of a distributed system.

## Todos
 - Try to write an Ansible Role.

## License
[Ansible Docker etcd Cluster](https://github.com/yungshun317/ansible-docker-etcd-cluster) is released under the [MIT License](https://opensource.org/licenses/MIT) by [yungshun317](https://github.com/yungshun317).
