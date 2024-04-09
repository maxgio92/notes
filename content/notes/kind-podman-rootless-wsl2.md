---
Title: Kind with Podman rootless mode on WSL2
---

sudo apt update
Install podman
```shell
sudo apt install -y podman
```

Configure podman /etc/containers/containers.conf
```shell
log_driver = "k8s-file"
events_logger = "file"
cgroup_manager = "cgroupfs"
```

Install kind

Disable cgroupv1 for the WSL instance (or globally at \Users\${USER}\.wslconfig) , for all cgroup controllers - ref: https://devpress.csdn.net/postgresql/630fab296a097251580cf46e.html

```shell
cat <<EOF >/etc/wsl.conf
[wsl2]
kernelCommandLine = cgroup_no_v1=all
EOF
```

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

Install missing CNI plugins v1.1.1 - ref: https://bugs.launchpad.net/ubuntu/+source/libpod/+bug/2024394
```
curl -SLO http://archive.ubuntu.com/ubuntu/pool/universe/g/golang-github-containernetworking-plugins/containernetworking-plugins_1.1.1+ds1-3_amd64.deb
sudo dpkg -i containernetworking-plugins_1.1.1+ds1-3_amd64.deb
```

Upgrade crun >= 1.9.0 - ref: https://noobient.com/2023/11/15/fixing-ubuntu-containers-failing-to-start-with-systemd/
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

Configure systemd resource delegation - ref: https://unix.stackexchange.com/questions/624428/cgroups-v2-cgroup-controllers-not-delegated-to-non-privileged-users-on-centos-s/625079#625079

/etc/systemd/system/user-0.slice:
```
[Unit]
Before=systemd-logind.service
[Slice]
Slice=user.slice
[Install]
WantedBy=multi-user.target
```

/etc/systemd/system/user@.service.d/delegate.conf:
```
[Service]
Delegate=cpu cpuset io memory pids
```

/etc/systemd/system/user-.slice.d/override.conf:
```
[Slice]
Slice=user.slice

CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksAccounting=yes
```
