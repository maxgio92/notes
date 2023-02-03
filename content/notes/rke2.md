---
Title: RKE2 Operations
---

# RKE2

Secuirty and compliance:
- hardened to pass CIS K8s benchmark
- enabled FIPS 140-2
- regularly scans componets images with trivy

Flavours of k8s:
- RKE (Docker as container engine, traditional)
- RKE 2 (hardened, next generation)
- K3S (focused on IoT, lightweight)

K3S + RKE + security = RKE 2

RKE 2 does not rely on Docker
RKE 2 launches CP compoents as static pods (managed by Kubelet)

## RKE2 and K3s

RKE2 is based on K3s.
Let's see the small differences.

### RKE2

Data store: etcd.
It provides node-token at `/var/lib/rancher/rke2/server/node-token` for additional nodes.
Cluster configuartion file is expected at `/etc/rancher/rke2/config.yaml`.

Intallation steps are nearly identical.
```
curl -sL https://get.rke2.io | sh -
systemctl enable --now rke2-server
```

### K3s

Data store: SQLite.
It provides node-token at `/etc/rancher/node/password` for additional nodes.
Cluster configuartion file is expected at `/etc/rancher/k3s/config.yaml`.

Intallation steps are nearly identical:
```
curl -sL https://get.k3s.io | sh -
systemctl enable --now k3s-server
```

## Deploy

Deploy the script
ENable and start systemd svc


```
curl -sL https://get-rke2.io | sudo sh -
# stable Rke version is downloaded with related systemd services
# scripts installed under /usr/local/bin

systemctl enable --now rke2-server.service

# check the journal for the unit
journalctl -u rke2-server -f
# creates a new kubeconfig file at /etc/rancher/rke2/rke2.yaml (root owned)
# 5~ mins
```

## Requirements

- Two nodes same hostnames shoudl be avoided
  - If any, `node-name` parameter in the RKE2 `config.yaml` must have different values
- OS:
  - SUSE (amd64)
  - Ubuntu 18.04, Ubuntu 20.04 (amd64)
  - Centos/RHEL 7.8, Centos/RHEL 8.2 (amd64)
- 512 MB min (1GB min recommended)
- 1 CPU min
- Ports:
  - Server:
    - 6443 TCP from agents
    - 9345 TCP from agents
    - 2379 TCP etcd client, from servers
    - 2380 TCP etcd peer, from servers
  - Server and Agents:
    - 8472 UDP from all (only Flannel VXLAN, which is the standard overlay for RKE2 - should NOT be exposed externally)
    - 10250 TCP Kubelet
    - 30000-32767 NodePort range, from all
  - more on `https://docs.rke2.io/install/network_options/`

## Nodes

- Server nodes: etcd, apiserver, it can be a cp, etcd or worker node (when not Tainted)
- Agent nodes: only run user apps

##  RKE2 architecture

It's based on K3s, it leverages the same simple installation process.
A single binary is installed on all nodes which are members of the cluster.

```
User (kubectl) > Server nodes > Agent nodes
```

### Data store

Same architecture of K3s, the only difference is etcd instead of sqlite.

## Configuration

It needs to be done before systemd services start.
It can be done via:
- CLI flags
- `config.yaml` file

### CLI

```
rke2 server \
--write-kubeconfig "0644" \
--tls-san "foo.local" \
--node-label "foo=bar"
```

> Note: it's not run with systemd.

```
cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
tls-san:
- "foo.local"
node-label:
- "foo=bar"
EOF
```

Whenever new cluster is created, it's configured as declared above.

More:
- `https://docs.rke2.io/install/install_options/install_options`
- `https://docs.rke2.io/install/install_options/agent_config`
- `https://docs.rke2.io/install/install_options/server_config`.

### Creating config.yaml for Server nodes

For the seed Server node:

```
sudo mkdir -p /etc/rancher/rke2
sudo cp config.yaml /etc/rancher/rke2/config.yaml
cat <<EOF > /etc/rancher/rke2/config.yaml
node-name:
- "cplane01"
node-taint:
- "CriticalAddonsOnly=true:NoExecute"
tls-san:
- loadbalancer.example.com # especially for HA
EOF
```

> This is the same for all the nodes

For additional Server nodes:

```
cat <<EOF > /etc/rancher/rke2/config.yaml
server:
- "https://cplane01.example.com:9345"
token:
- "k105bd..."
node-name:
- "cplane02"
node-taint:
- "CriticalAddonsOnly=true:NoExecute"
EOF
```

### Creating cofig.yaml for Agent nodes

Get the `node-token` from the first Server node:

```
sudo cat /var/lib/rancher/rke2/server/node-token
```

Configure the `config.yaml` file:

```
cat <<EOF > /etc/rancher/rke2/config.yaml
server:
- "https:/cplane01.example.com:9345"
token:
- "k105bd..."
node-name:
- "worker01"
node-label:
- "node=worker"
EOF
```

## Installation

The installation script `https://get.rke2.io` can be run before or after creating the `cluster.yaml` (`config.yaml`?).
The script creates the following:
- `/usr/local/bin/rke2`
- `/usr/local/bin/rke2-killall.sh`
- `/usr/local/bin/rke2-uninstall.sh`
- `/var/lib/rancher/rke2/server/node-token`
- systemd services for `rke2-agent` and `rke2-server`

Then:

```
sudo systemctl enable --now rke2-server
sudo journalctl -u rke2-server -f
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/rke2.kubeconfig`
chown $(id -u):$(id -g) ~/.kube/rke2.kubeconfig
```

## Additional options

- RPM files for deploying the inistial scripts are available for some Linux distribution
- `/usr/local/bin/rke2` can be run outside of systemd (e.g. to create clusters, not recommeded)

## Deploy a Server node

```
sudo -i
curl -sL https://get-rke2.io | sh -
cat <<EOF > /etc/rancher/rke2/config.yaml
node-name:
- cplane01
tls-san:
- lb.example.com
node-label
- "node=notaint"
EOF
```

Configurations in `cluster.yaml` (`config.yaml`?) must exist before the below:

```
systemctl enable --noe rke2-server.service
journalctl -fu rke2-server
# ...

exit
mkdir ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/
chown $(id -u):$(id -g) ~/.kube/rke2.yaml
kubectl get nodes -w
kubectl get pods -A -w
# ...

```

## Deploying additional Server nodes

A LB is a good choice for HA, whne adding Server nodes (L4 LB, round-robin DNS, etc.).

Edit kubeconfig changing the `cluster.server` field to the LB endpoint.

## Deploying Agent nodes

Get the node-token from first the Server node.

```
sudo cat /var/lkib/rancher/rke2/server/node-token
```

and put it in the `config.yaml` file for the Agent:

```
sudo -i
mkdir -p /etc/rancher/rke2
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://cplane01.example.com:9345
token: <NODE_TOKEN>
EOF
curl -sL https://get-rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

Configurations in `cluster.yaml` / (`config.yaml`?) must exist before starting the systemd service:

```
systemctl enable --now rke2-agent.service
```

## Troubleshooting

- `config.yaml` syntax
- Starting nodes before ready: if the systemd service on a node is started before `config.yaml` is ready, then the node will need to have RKE2 reinstalled before progressing.

### Uninstalling RKE2 on a Node

```
/usr/local/bin/rke2-uninstall.sh
```

> Installed when downloading RKE2.

When a single node cluster, the whole cluster is deleted, otherwise only the current node.
This is *irreversible*.
