# OKD Agent-Based Install Guide

## Requirements

- **Bastion Host:** Small VM to run `oc` and OpenShift Installer commands.
- **Minimum OKD Node:**  
  - x86_64 CPU  
  - 16GB RAM  
  - 8 CPU cores  
  - 150GB disk space

## Steps

### 1. Prepare Configuration Files

Create a working directory and edit your config files:

```sh
mkdir okd-install && cd okd-install
vi agent-config.yaml
vi install-agent.config
```

#### Example `agent-config.yaml`

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: okd
rendezvousIP: 192.168.68.201
hosts:
  - hostname: okd
    role: master
    rootDeviceHints:
      deviceName: "/dev/nvme0n1"
    interfaces:
      - name: enp2s0
        macAddress: "58:47:ca:7e:54:f0"
    networkConfig:
      interfaces:
        - name: enp2s0
          type: ethernet
          state: up
          ethernet: {}
          ipv4:
            enabled: true
            address:
              - ip: 192.168.68.201
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.68.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.68.1
            next-hop-interface: enp2s0
            table-id: 254
additionalNTPSources:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
```

#### Example `install-agent.config`

```yaml
apiVersion: v1
baseDomain: home.local
metadata:
  name: okd
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.68.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths": {"empty": {"auth": "ZW1wdHk6ZW1wdHkK"}}}'  # Bogus Pull Secret per OKD Docs
sshKey: ssh-ed25519 <Your Key> <Comment>
```

### 2. Extract the OpenShift Installer

```sh
oc adm release extract --command=openshift-install quay.io/okd/scos-release:4.19.0-okd-scos.15
```

### 3. Create the ISO Image

```sh
./openshift-install agent create image --dir . --log-level=debug
```

### 4. Write ISO to USB Device

```sh
sudo dd if=agent.x86_64.iso of=/dev/<YourUSBDevice> bs=4m status=progress
```

### 5. Boot and Install OKD Cluster

- Boot your machine or VM from the USB stick or ISO.
- On your bastion host, run:

```sh
./openshift-install agent wait-for install-complete --dir=. --log-level=debug
```

Follow onscreen instructions to access your cluster.

---
