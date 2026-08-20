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

**Under the hood:** these four directories aren't arbitrary — they're exactly
the inputs the OverlayFS kernel driver expects, and each one plays a
different role once step 3 mounts them together. It's worth understanding
each in isolation *before* anything is mounted, since the mount step (3)
is where they start interacting:

- **`lower/`** — the read-only base layer. This is where the Alpine rootfs
  tarball gets extracted in the next step. OverlayFS treats this directory
  as immutable: no matter what happens inside the running container, the
  kernel driver will never write to `lower/`. In container terminology this
  *is* the "image" — a pristine, reusable starting point.
- **`upper/`** — the writable layer. Once mounted, every file the container
  creates, modifies, or deletes shows up here, and only here. This is what
  makes the base image reusable across runs: you can blow away `upper/` and
  `work/` and remount against the same untouched `lower/` to get a fresh
  container, instead of re-extracting the whole rootfs each time.
- **`work/`** — internal bookkeeping space the kernel driver requires to
  perform certain operations (like promoting a file from `lower/` into
  `upper/`) atomically, using rename-based tricks under the hood. It has to
  live on the *same filesystem* as `upper/`, has to start out empty, and you
  should never read from or write to it directly — nothing you care about is
  stored there; it's purely the driver's scratch space.
- **`merged/`** — the union mountpoint. Once step 3 mounts the overlay here,
  this directory transparently presents `lower/` and `upper/` combined as a
  single filesystem tree. This is the directory that later gets `chroot`'d
  into (step 5) and effectively *becomes* the container's `/`.

This exact four-directory split — lower, upper, work, merged — is not a
simplification invented for this lab; it's literally what Docker's
`overlay2` storage driver (and most other container runtimes) does under
the hood to implement image layers and container-writable layers.

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

**Under the hood:** this step is doing exactly what pulling and unpacking an
image layer does in a real container runtime — you're populating `lower/`,
the read-only base directory from step 1, with actual file content. Nothing
overlay-specific happens yet (that starts at the mount in step 3); this is
just `tar` extracting files onto disk.

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

**Under the hood:** this is where the four directories from step 1 actually
start working together, driven by the OverlayFS kernel module:

- **Lookup order.** When something reads a path under `merged/`, the kernel
  checks `upper/` first; if the file isn't there, it falls through to
  `lower/`. The result is a single merged view even though the files
  physically live in two separate directories.
- **Copy-up.** The first time a *write* touches a file that currently only
  exists in `lower/`, the driver doesn't modify `lower/` in place — instead
  it copies that file into `upper/` (using `work/` internally to do the copy
  + rename atomically) and then applies the write to the copy. This is the
  mechanism that guarantees `lower/` never changes no matter what the
  container does to its files.
- **Whiteouts.** Deleting a file that only exists in `lower/` can't actually
  remove it (it's read-only), so the driver creates a special marker in
  `upper/` — a character device inode with major/minor number 0:0, called a
  "whiteout" — that tells the merged view "treat this path as gone," even
  though the real file is still sitting untouched in `lower/`.
- **Opaque directories.** A similar trick (an extended attribute on the
  directory) lets `upper/` fully replace a directory from `lower/` instead
  of merging its contents.

The net effect: `merged/` looks and behaves like a complete, ordinary
filesystem to anything that reads from it (including the `chroot` in step 5),
even though physically almost nothing was copied — only `lower/`'s
directory structure is being reused, and `upper/` only ever holds the delta.

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

**Under the hood:** `/sys/fs/cgroup` isn't a normal directory tree — it's
`cgroupfs`, a virtual filesystem the kernel exposes so cgroup management can
be done with plain `mkdir`/`echo`/`cat` instead of dedicated syscalls.
Creating a directory here creates a real kernel cgroup object; `memory.max`
is read by the kernel's memory controller, which tracks page allocations
against that limit and triggers the OOM killer for any process in the
cgroup once it's exceeded. This VM uses cgroup v2's *unified* hierarchy (one
single tree covering all controllers — memory, cpu, pids, etc.), unlike the
older v1 model where each controller had its own separate tree. The cgroup
is created empty on purpose: `cgroup.procs` only lists member PIDs, and none
exist yet.

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

**Under the hood:** each `unshare` flag detaches the new process from one
piece of shared kernel state and gives it a private copy:

- `--pid` (`CLONE_NEWPID`) — a new PID namespace. The process becomes PID 1
  *inside* that namespace (that's why `ps aux` shows almost nothing — it's
  a fresh process tree), but it still has a real, ordinary PID as seen from
  the host (visible via `/proc/<pid>/status`'s `NSpid` field) — that's the
  PID Shell B uses in step 6 and 7.
- `--uts` (`CLONE_NEWUTS`) — a private hostname/domainname, so `hostname
  containerdemo` doesn't affect the VM.
- `--ipc` (`CLONE_NEWIPC`) — isolated System V IPC objects and POSIX message
  queues.
- `--mount` (`CLONE_NEWNS`) — a private copy of the mount table, so mounting
  `/proc` and `/sys` inside the container doesn't touch the VM's own mounts.
- `--net` (`CLONE_NEWNET`) — a private network stack: its own interfaces,
  routing table, and iptables rules. This is why `ip link` shows nothing
  usable yet — a new net namespace starts completely empty, not even with
  loopback enabled (that's fixed in step 7).
- `--mount-proc` remounts `/proc` inside the new mount+PID namespace so it
  reflects *this* process tree instead of leaking the host's.

`chroot` itself only changes which directory syscalls treat as `/` for path
lookups — it's a much older, weaker mechanism than the namespaces above, and
by itself it's not a real security boundary (a privileged process can escape
a bare chroot). Real container runtimes use `pivot_root` instead, which
fully swaps the root filesystem including the old root out of reach. This
lab uses `chroot` because it's simpler to reason about for learning
purposes — worth keeping in mind if you ever compare this to production
runtime internals.

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

**Under the hood:** a process can't cleanly move itself into a cgroup before
it's finished being created, so this has to happen from the outside, after
the fact. Writing a PID to `cgroup.procs` updates that task's `css_set`
pointer inside the kernel — the internal structure that ties a process to
its cgroup membership across all controllers. From that point on, any memory
the process (or anything it later forks) allocates is charged against this
cgroup's `memory.max`, and children automatically inherit their parent's
cgroup membership.

## 7. Networking: veth pair + NAT

**Shell B.** A fresh network namespace starts with zero interfaces (not even
loopback up). We bridge it to the host with a virtual ethernet pair — one end
stays on the host, the other moves into the container's netns.

```bash
# Find the container's PID again if you don't still have it
CPID=<PID>

# The VM's internet-facing interface (VirtualBox NAT adapter). On this box
# it's enp0s3 — confirmed via `ip route` (the `default via ...` line).
UPLINK=$(ip route show default | awk '{print $5; exit}')
echo "$UPLINK"

ip link add veth-host type veth peer name veth-ctr
ip link set veth-ctr netns $CPID

# Host side
ip addr add 10.200.1.1/24 dev veth-host
ip link set veth-host up

# Enable forwarding + NAT so the container can reach the internet
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o "$UPLINK" -j MASQUERADE
iptables -A FORWARD -i veth-host -o "$UPLINK" -j ACCEPT
iptables -A FORWARD -i "$UPLINK" -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT
```

> `$UPLINK` resolves to the VM's default-route interface — on the
> `ubuntu/jammy64` box used by this lab's `Vagrantfile` that's `enp0s3`
> (VirtualBox's NAT adapter, `10.0.2.x`). The `private_network` adapter shows
> up separately as `enp0s8` (`192.168.56.x`) and isn't the internet uplink.

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

**Under the hood:**

- **veth pair.** `veth-host`/`veth-ctr` are a *virtual* Ethernet cable: a
  packet written to one end is handed directly to the other end's receive
  path by the kernel's veth driver — there's no real wire, no switch, no
  physical layer at all, just an in-kernel handoff.
- **`ip link set veth-ctr netns $CPID`** reparents that `net_device` out of
  the host's network namespace into the container's — the interface
  physically "moves," it isn't cloned. That's why it only shows up on one
  side at a time.
- **Why nothing worked before this step:** a brand-new net namespace (from
  `--net` in step 5) starts with zero interfaces, not even loopback enabled
  — it is genuinely empty until something like this wires it up.
- **`net.ipv4.ip_forward=1`** flips on the kernel's IP forwarding path,
  letting it route packets *between* interfaces (veth-host and the uplink)
  instead of only accepting traffic addressed to itself.
- **`MASQUERADE`** is a netfilter rule in the `nat` table's `POSTROUTING`
  hook: it rewrites the container's private source IP (`10.200.1.2`) to the
  VM's own uplink address as packets leave, and — using connection tracking
  (conntrack) — automatically reverses that rewrite on the way back, so
  replies find their way back to the container.
- **The `FORWARD` rules** are the netfilter gate that decides whether
  packets are even allowed to move between `veth-host` and the uplink
  interface at all; without them, forwarding is enabled but nothing passes.

## 8. Install and run nginx inside the container

**Shell A.**

```bash
apk update
apk add nginx
nginx
curl 127.0.0.1               # should return the nginx welcome page
```

**Under the hood:** `apk` reaching Alpine's package mirror is the first real
proof that step 7's plumbing works — it needs both a route out (the veth +
NAT path) and working DNS (the `/etc/resolv.conf` written in step 7) inside
what is, from the kernel's point of view, a fully independent network stack.

## 9. Reach the container from the host

**Shell B.**

```bash
curl 10.200.1.2
```

If this returns the same nginx welcome HTML, the full path — namespaces,
veth, routing, NAT/forwarding rules — is working end to end.

**Under the hood:** trace the packet in the opposite direction from step 7.
The VM's normal routing table sends this request out `veth-host`; the veth
driver hands it directly to `veth-ctr` on the other end — inside the
container's netns — with no involvement of the uplink/NAT path at all
(that machinery is only needed for the container reaching *outward*, not
for the host reaching *in* over the veth link). The container's own,
independent TCP/IP stack then delivers the connection to nginx's listening
socket, exactly as it would for any other client.

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

**Under the hood:** this is copy-up from step 3 made visible. Every file
`apk add nginx` installed and every file nginx wrote or touched while
running triggered a copy-up into `upper/` (or, for anything nginx deleted,
left a whiteout marker there) — `lower/` is byte-for-byte the same Alpine
tarball you extracted in step 2. Diffing `lower/` against `merged/` is
exactly the algorithm real container tooling uses to compute a layer's diff
(`docker diff`) or to squash/export an image layer — you're looking at the
same upper-directory contents a runtime would package up as "what this
container changed."

## 11. Teardown

**Shell A** — exit the container:

```bash
exit        # leaves the unshare'd shell; its namespaces are torn down
```

**Shell B** — clean up the host-side state:

```bash
iptables -t nat -D POSTROUTING -o "$UPLINK" -j MASQUERADE
iptables -D FORWARD -i veth-host -o "$UPLINK" -j ACCEPT
iptables -D FORWARD -i "$UPLINK" -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT
ip link delete veth-host 2>/dev/null || true   # also removes veth-ctr, the pair is linked
umount /containerdemo/merged
rmdir /sys/fs/cgroup/containerdemo
```

**Under the hood:** namespaces are reference-counted kernel objects, not
files you delete — a namespace is only actually freed once the last process
inside it exits *and* nothing else (an open file descriptor, a bind mount)
still references it, which is why simply exiting the `unshare`d shell is
enough to tear down all five namespaces it created. `veth-host` and
`veth-ctr` are a linked pair, so deleting either end automatically removes
the other — there's no separate cleanup step for the container side.
`rmdir` on the cgroup only succeeds because it's empty of processes (the
container's shell already exited in the previous command); a cgroup with
live members can't be removed. `umount` simply releases the overlay
filesystem's in-kernel superblock — `lower/` and `upper/` themselves are
untouched on disk and can be remounted later.

Or, for a fully clean slate, just discard the whole VM:

```bash
vagrant destroy -f
```

and re-run `vagrant up` to start the lab over from scratch.
