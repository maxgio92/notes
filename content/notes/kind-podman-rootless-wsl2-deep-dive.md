---
Title: Kind with Podman rootless mode on WSL2 and deep dive
---

**Enable systemd on the WSL machine**

```shell
cat <<EOF >>/etc/wsl.conf
[boot]
systemd=true
EOF
```

Disable cgroupv1 for the WSL instance (or globally at \Users\%USERPROFILE%\.wslconfig), for all cgroup controllers:

```shell
cat <<EOF >>/etc/wsl.conf
[wsl2]
kernelCommandLine = cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1
EOF
```
> References:
> - https://docs.kernel.org/admin-guide/cgroup-v2.html
> - https://devpress.csdn.net/postgresql/630fab296a097251580cf46e.html
> - https://github.com/spurin/wsl-cgroupsv2

From Windows host, restart WSL
```shell
wsl --shutdown
```

Move the cgroupv2 mount to classic /sys/fs/cgroup, for podman/docker clients
```shell
cat <<EOF >>/etc/fstab
cgroup2 /sys/fs/cgroup cgroup2 rw,nosuid,nodev,noexec,relatime,nsdelegate 0 0
EOF
mount -a
```

**Podman**

Install Podman. Depending on the Linux distribution the command changes. For example for Ubuntu:

```shell
sudo apt update
sudo apt install -y podman
```

As journald is not available in WSL, configure podman's log driver, events logger.
Also, set the cgroup manager to `cgroupfs`:

```shell
cat <<EOF >/etc/containers/containers.conf
log_driver = "k8s-file"
events_logger = "file"
cgroup_manager = "cgroupfs"
EOF
```

Install missing CNI plugins v1.1.1
```
curl -SLO http://archive.ubuntu.com/ubuntu/pool/universe/g/golang-github-containernetworking-plugins/containernetworking-plugins_1.1.1+ds1-3_amd64.deb
sudo dpkg -i containernetworking-plugins_1.1.1+ds1-3_amd64.deb
```

> Reference: https://bugs.launchpad.net/ubuntu/+source/libpod/+bug/2024394

Upgrade `crun`, version >= 1.9.0:

```shell
mkdir -p ~/.local/bin
curl -sL "https://github.com/containers/crun/releases/download/${CRUN_VER}/crun-${CRUN_VER}-linux-amd64" -o "${HOME}/.local/bin/crun"
chmod +x ~/.local/bin/crun
mkdir -p ~/.config/containers"
cat << EOF > ~/.config/containers/containers.conf
[engine.runtimes]
crun = [
  "${HOME}/.local/bin/crun",
  "/usr/bin/crun"
]
EOF
podman info | grep -i crun
```

> Reference: https://noobient.com/2023/11/15/fixing-ubuntu-containers-failing-to-start-with-systemd/

**Configure systemd resource delegation, to run Podman in rootless mode**

For root user slice:

```shell
cat <<EOF >/etc/systemd/system/user-0.slice
[Unit]
Before=systemd-logind.service
[Slice]
Slice=user.slice
[Install]
WantedBy=multi-user.target
EOF
```

For all other user slices:

```shell
cat <<EOF >/etc/systemd/system/user-.slice.d/override.conf
[Slice]
Slice=user.slice

CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksAccounting=yes
EOF
```

For all user manager instances:

```
cat <<EOF >/etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=cpu cpuset io memory pids
EOF
```

> References:
> - https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html
> - https://unix.stackexchange.com/questions/624428/cgroups-v2-cgroup-controllers-not-delegated-to-non-privileged-users-on-centos-s/625079#625079

Note: `Delegate=yes` delegate all supported controllers.

**Create a KinD cluster**

Install kind [the way](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) you prefer.

Now run the `kind` command to create cluster containers as user slices, instead of system ones (thus in cgroups under the Systemd user slice's cgroup):

```shell
systemd-run --scope --user kind create cluster
```

> Reference: https://kind.sigs.k8s.io/docs/user/rootless/#creating-a-kind-cluster-with-rootless-podman

Validate the control plane container:

```shell
$ kubectl version
Client Version: v1.29.3
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.29.2
$ systemd-run --scope --user podman exec -it kind-control-plane bash
# ps -C kube-apiserver
    PID TTY          TIME CMD
    565 ?        00:01:23 kube-apiserver
```

**Deep dive**

Taking a look at what is going on under the hood, it can be noted that Podman created the KinD control-plane container's `cgroup` (`a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38` in the example below).

Te container, the `init.scope` and the `kubelet.service` cgroups can be noted:

```shell
$ systemctl status
● mywsl
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Tue 2024-04-09 14:34:18 CEST; 4h 7min ago
   CGroup: /
           ├─user.slice
           │ └─user-1000.slice
           │   ├─user@1000.service
           │   │ ├─app.slice
           │   │ │ ├─podman.service
           │   │ │ │ ├─532 /usr/bin/podman
           │   │ │ │ └─734 /usr/bin/dbus-daemon --syslog --fork --print-pid 4 --print-address 6 --session
           │   │ │ ├─a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38
           │   │ │ │ ├─init.scope
           │   │ │ │ │ └─70678 /sbin/init
           │   │ │ │ ├─system.slice
           │   │ │ │ │ ├─containerd.service
...
           │   │ │ │ └─kubelet.slice
           │   │ │ │   ├─kubelet.service
           │   │ │ │   │ └─72203 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///run/con
...
```

By comparison instead, this is similar hierarchy but with rooful Docker-based KinD container:

```
$ systemctl status
● brodequin
    State: running
    Units: 401 loaded (incl. loaded aliases)
     Jobs: 0 queued
   Failed: 0 units
    Since: Fri 2024-04-05 18:56:16 CEST; 3 days ago
  systemd: 255.4-2-arch
   CGroup: /
           ├─init.scope
           │ └─1 /sbin/init
           ├─system.slice
           │ ├─containerd.service
...
           │ ├─docker-8aea66260a20e4fe20c03c3424d4f294b56eab451430492123e736c3f749ebca.scope
           │ │ ├─init.scope
           │ │ │ └─43996 /sbin/init
           │ │ ├─kubelet.slice
           │ │ │ └─kubelet.service
           │ │ │   └─44913 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///run/containerd/containerd.sock --node-ip=172.18.0.2 --node-l>
           │ │ └─system.slice
           │ │   ├─containerd.service
...
           │ │   └─systemd-journald.service
           │ │     └─44127 /lib/systemd/systemd-journald
           │ ├─docker.service
           │ │ ├─ 1368 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
           │ │ └─43961 /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 40721 -container-ip 172.18.0.2 -container-port 6443
```

From the KinD control plane container it can be seen that is a normal _domain_ `cgroup` and no processes are assigned to it:

```shell
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/
a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38/cgroup.type
domain
cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38/cgroup.procs
```

However, under it a whole system.slice is created with /sbin/init _scope_ cgroup and a cgroup for Containerd:

```shell
$ ls -l /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/
a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38/system.slice
cgroup.controllers
cgroup.events
...
containerd.service
...
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/
a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38/system.slice/containerd.service/cgroup.type
domain
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/
a4cfc6bc8a38ee23e42839393eedba68a92f7f8b3a23e70328a29e58c0c59d38/system.slice/containerd.service/cgroup.procs
70837
71711
71731
71755
71798
73781
73816
74237
74259
74284
```

They're exactly the Kubernetes pods' processes, plus Containerd itself.

For example:

```shell
$ ps -q 74284 -o args
COMMAND
/usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 4796ab87414f85c7accdb5c7c858e37121f8dcfd0cd483fb74233d25a8f06817 -address /run/containerd/containerd.sock
```

and it can be noted that is related to one pod:

```shell
$ systemd-run --scope --user podman exec -it kind-control-plane crictl ps --pod 4796ab87414f85c7accdb5c7c858e37121f8dcfd0cd483fb74233d25a8f06817
Running scope as unit: run-r3403981399e3484c92993c384f192d63.scope
CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID              POD
0a98af2598be4       0500518ebaa68       36 minutes ago      Running             local-path-provisioner   0                   4796ab87414f8       local-path-provisioner-7577fdbbfb-zjdbq
```

Trying to run a rootful standard Nginx pod, it can be noted that the root user that runs the Nginx daemon in the container, is mapped to the UID 1000 in the root namespace, looking outside of the container:

```shell
$ kubectl run --image nginx nginx
pod/nginx created
$ ps -C nginx -o user -o args n
    USER COMMAND
    1000 nginx: master process nginx -g daemon off;
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
  100100 nginx: worker process
```

Indeed, Podman creates a dedicated user namespace for containers:

```
$ lsns --type user
        NS TYPE  NPROCS   PID USER  COMMAND
4026531837 user      10   260 massi /lib/systemd/systemd --user
4026532263 user      90   532 massi /usr/bin/podman
```

That's it.

---

Documentation:
- https://www.freedesktop.org/software/systemd/man/latest/user@.service.html
- https://www.freedesktop.org/software/systemd/man/latest/systemd.slice.html
- https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html
- https://docs.kernel.org/admin-guide/cgroup-v2.html
- https://docs.podman.io/en/stable/markdown/podman.1.html
