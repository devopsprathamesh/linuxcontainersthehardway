# Linux containers the hard way — VM lab guide

This lab rebuilds a container from raw kernel primitives — no Docker,
containerd, or Podman — inside a disposable Vagrant VM, so you can break
things freely. It follows the same stages as the video: overlayfs, namespaces
+ chroot, cgroups, veth + NAT networking, and running nginx as proof it works.

## 0. Boot the VM

```bash
vagrant up
vagrant ssh
```

Everything below runs **inside the VM**. Become root for the whole session:

```bash
sudo -i
```

You'll need **two root shells** into the VM from step 7 onward — one stays
"inside" the container's namespaces, the other stays on the host to wire up
cgroups/networking from the outside. Open a second terminal now and run
`vagrant ssh` there too, then `sudo -i` in it as well, so it's ready when you
need it. Call this **Shell A** (will enter the container) and **Shell B**
(stays on the host).

---

## 1. Workspace layout

**Shell A.**

OverlayFS needs three directories plus a mountpoint:

- `lower/` — the read-only base image
- `upper/` — writable layer that captures every change
- `work/` — scratch space overlayfs requires internally (never touch it)
- `merged/` — the union view the container actually sees as `/`

```bash
mkdir -p /containerdemo/{lower,upper,work,merged}
cd /containerdemo
```

## 2. Fetch the Alpine minirootfs

**Shell A.**

Alpine publishes a tiny (~3MB) root filesystem tarball — enough to get a
shell, package manager, and eventually nginx.

```bash
curl -L -o alpine-minirootfs.tar.gz \
  https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs.tar.gz -C lower
ls lower
```

> If that exact version 404s, browse
> `https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/` and swap in
> the current filename.

## 3. Mount the overlay

**Shell A.**

```bash
mount -t overlay overlay \
  -o lowerdir=/containerdemo/lower,upperdir=/containerdemo/upper,workdir=/containerdemo/work \
  /containerdemo/merged

ls /containerdemo/merged
```

`merged` now shows Alpine's filesystem. Any file the container writes lands
in `upper` — `lower` stays untouched, which is what lets you diff the
container's footprint later (step 10).

## 4. Set up a cgroup (memory limit)

**Shell B** (host side — this cgroup will control a process we create next).

Ubuntu 22.04 mounts the unified cgroup v2 hierarchy at `/sys/fs/cgroup`.

```bash
mkdir /sys/fs/cgroup/containerdemo
echo 268435456 > /sys/fs/cgroup/containerdemo/memory.max   # 256MB cap
cat /sys/fs/cgroup/containerdemo/memory.max
```

The cgroup exists but has no processes yet — you'll add the container's PID
in step 6, once it exists.

## 5. Create namespaces + chroot

**Shell A.**

`unshare` detaches the new process from the host's PID/UTS/IPC/mount/network
namespaces before `chroot`-ing it into the overlay. `--mount-proc` gives it
its own `/proc` reflecting only its own (empty, so far) process tree.

```bash
unshare --pid --uts --ipc --mount --net --fork --mount-proc \
  chroot /containerdemo/merged /bin/sh
```

You're now inside the container. Confirm the isolation:

```bash
hostname containerdemo
hostname
ps aux                 # should show almost nothing — this is PID 1's own tree
ip link                # only shows nothing or a down 'lo' — no host interfaces visible
```

Mount `/sys` too (needed later for nginx and general sanity):

```bash
mount -t sysfs sysfs /sys
```

Leave this shell open and idle here — this is your container's live shell.

## 6. Attach the container to the cgroup

**Shell B.**

A brand-new process can't easily move itself into a cgroup before it exists,
so we do it from the outside. Find the PID of the `unshare`d process as seen
from the host:

```bash
ps aux | grep "[c]hroot /containerdemo/merged"
```

Take that PID and add it to the cgroup:

```bash
echo <PID> > /sys/fs/cgroup/containerdemo/cgroup.procs
cat /sys/fs/cgroup/containerdemo/cgroup.procs
```

The container process (and anything it forks) is now memory-capped.

## 7. Networking: veth pair + NAT

**Shell B.** A fresh network namespace starts with zero interfaces (not even
loopback up). We bridge it to the host with a virtual ethernet pair — one end
stays on the host, the other moves into the container's netns.

```bash
# Find the container's PID again if you don't still have it
CPID=<PID>

ip link add veth-host type veth peer name veth-ctr
ip link set veth-ctr netns $CPID

# Host side
ip addr add 10.200.1.1/24 dev veth-host
ip link set veth-host up

# Enable forwarding + NAT so the container can reach the internet
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i veth-host -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT
```

> `eth0` is the VM's NAT uplink from VirtualBox — confirm with `ip route` on
> the host if your interface is named differently.

**Shell A** (back inside the container):

```bash
ip link set lo up
ip addr add 10.200.1.2/24 dev veth-ctr
ip link set veth-ctr up
ip route add default via 10.200.1.1

# DNS so `apk` can resolve hostnames
echo "nameserver 8.8.8.8" > /etc/resolv.conf

ping -c 2 10.200.1.1        # host reachable
```

## 8. Install and run nginx inside the container

**Shell A.**

```bash
apk update
apk add nginx
nginx
curl 127.0.0.1               # should return the nginx welcome page
```

## 9. Reach the container from the host

**Shell B.**

```bash
curl 10.200.1.2
```

If this returns the same nginx welcome HTML, the full path — namespaces,
veth, routing, NAT/forwarding rules — is working end to end.

## 10. Inspect the overlay diff

**Shell B.**

Everything nginx's install and startup touched lives in `upper/`, while
`lower/` (the pristine Alpine base) is untouched:

```bash
find /containerdemo/upper -maxdepth 3
diff <(cd /containerdemo/lower && find . | sort) \
     <(cd /containerdemo/merged && find . | sort) | head -50
```

This is the same mechanism a real container runtime uses to build image
layers and compute `docker diff`-style output.

## 11. Teardown

**Shell A** — exit the container:

```bash
exit        # leaves the unshare'd shell; its namespaces are torn down
```

**Shell B** — clean up the host-side state:

```bash
iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
iptables -D FORWARD -i veth-host -o eth0 -j ACCEPT
iptables -D FORWARD -i eth0 -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT
ip link delete veth-host 2>/dev/null || true   # also removes veth-ctr, the pair is linked
umount /containerdemo/merged
rmdir /sys/fs/cgroup/containerdemo
```

Or, for a fully clean slate, just discard the whole VM:

```bash
vagrant destroy -f
```

and re-run `vagrant up` to start the lab over from scratch.
