# Linux containers the hard way — VM lab guide

A container is not a special kernel object. There is no `struct container`
anywhere in the Linux kernel source. A "container" is just an ordinary
process that the kernel has been convinced to see a different, restricted
version of reality: a different root filesystem, a private view of process
IDs and hostnames, a private network stack, and a resource budget it can't
exceed. Docker, containerd, and Podman are, at their core, orchestration
layers around exactly four kernel mechanisms — nothing more:

1. **OverlayFS** — gives the process a filesystem that looks like a full
   Linux install, built cheaply on top of a shared read-only base.
2. **Namespaces** — give the process its own private *views* of PIDs,
   hostname, IPC, mounts, and network, so it can't see or interfere with
   anything outside itself.
3. **cgroups** — cap how much CPU/memory/etc. the process (and anything it
   spawns) is allowed to consume.
4. **chroot/pivot_root** — pin the process's filesystem root to the overlay
   from (1) so it can't walk back out to the host's real filesystem.

This lab builds all four by hand, one syscall/tool at a time, inside a
disposable Vagrant VM (so if you break namespaces, cgroups, or iptables
rules on the "host" — which here is just the VM — nothing on your real
machine is at risk; `vagrant destroy` gets you a clean kernel again in
seconds). Each step below explains **why that step exists** in the bigger
picture, then walks every command in a **before → what happened → after**
table, so you can see exactly what changed on the system at each line —
the goal is that by step 11 you could explain to someone else, from memory,
what `docker run` is actually doing under the hood.

## Prerequisites

This is not a "copy-paste and it works" lab — it's meant to be read and
understood, and the "Under the hood" sections lean on Linux fundamentals
without re-teaching them from scratch. Before starting, you should already
be comfortable with:

- **Basic Linux CLI usage** — navigating with `cd`/`ls`, editing/creating
  files, redirection (`>`, `>>`), and reading man pages when a flag is
  unfamiliar.
- **Processes and PIDs** — what a process is, what `fork()`/`exec()` roughly
  do, and how to read `ps`/`ps aux` output.
- **The Linux filesystem tree** — what `/`, `/etc`, `/proc`, `/sys` are for,
  and roughly what `mount`/`umount` do.
- **Basic networking** — IP addresses, subnets/CIDR notation (`/24`), what a
  default gateway and a routing table are, and roughly what NAT does.
- **`sudo`/root privileges** — everything here runs as root because
  namespaces, cgroups, and raw networking require it; you should understand
  *why* that's the case, not just that it is.

If any of those are shaky, this lab will still run if you copy the commands
verbatim, but the "Why this step exists" / "Under the hood" explanations —
which are the actual point of the exercise — will be much harder to follow.
Skimming a Linux fundamentals primer (processes, filesystems, basic
networking) first is strongly recommended over starting cold.

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

Until step 5, Shell A and Shell B are functionally identical — both are
plain root shells sitting in the VM's default (un-namespaced) PID, mount,
and network context. They only start to diverge once step 5's `unshare`
puts Shell A into a separate world.

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

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `mkdir -p /containerdemo/{lower,upper,work,merged}` | Shell A, VM's normal (default) filesystem — no namespaces active yet | `/containerdemo` doesn't exist at all | The shell expands the `{...}` brace list *itself* before running anything, turning this into one `mkdir` call with five paths; the kernel's `mkdir(2)` syscall creates the parent directory and all four children on the VM's real root filesystem (ext4, from the `ubuntu/jammy64` box) | `ls -la /containerdemo` shows `lower/ upper/ work/ merged/`, all empty, all real directories on disk |
| `cd /containerdemo` | Shell A | Shell's current working directory was wherever `sudo -i` started (typically `/root`) | A shell builtin, not even a syscall you'd strace as a separate program — calls `chdir(2)` to update *this shell process's* notion of cwd only; no kernel object is created and nothing else on the system is affected | `pwd` now prints `/containerdemo` |

**Why this step exists:** before we can talk about namespaces, cgroups, or
networking, the container needs *somewhere to live* — a root filesystem.
Real container images are distributed as layered tarballs precisely so that
many containers can share one base image without copying it per-container.
We're about to reproduce that layering scheme by hand, so before touching
overlayfs itself, we need the raw ingredients laid out on disk.

**Under the hood — think of OverlayFS like transparent sheets on an
overhead projector:**

- **`lower/`** — the bottom sheet, printed and permanent. This is where the
  Alpine rootfs tarball gets extracted in the next step. OverlayFS treats
  this directory as immutable — the kernel driver will *never* write to it,
  no matter what happens inside the running container. In container
  terminology this sheet *is* the "image": a pristine, reusable starting
  point that, in a real system, could be the same physical bytes shared
  read-only by a dozen different running containers simultaneously (this is
  exactly the copy-on-write efficiency trick that makes `docker run` on a
  cached image nearly instant — no bytes are copied at container start,
  only at mount time, and even then only metadata).
- **`upper/`** — a blank transparency sheet you lay on top and write on with
  a marker. Once mounted, every file the container creates, modifies, or
  deletes shows up here, and *only* here. This is what makes the base image
  reusable across runs: blow away `upper/` and `work/`, remount against the
  same untouched `lower/`, and you get a brand-new container with zero
  history, without re-extracting the whole rootfs. It's also precisely what
  `docker commit` reads from when it turns a running container into a new
  image layer, and what disappears forever when you `docker rm` a container
  without committing it.
- **`work/`** — not a sheet at all; it's the overlay driver's internal
  workbench, required so certain operations (specifically, promoting a file
  from `lower/` up into `upper/`, called "copy-up" — see step 3) can happen
  *atomically*. The trick the kernel uses is a full write to a hidden file
  in `work/`, followed by an atomic `rename(2)` into `upper/`. Because
  `rename(2)` is atomic at the filesystem level, any process reading that
  file mid-copy-up sees either the fully-old version or the fully-new
  version — never a half-written one, even if the machine crashes at the
  worst possible moment. `work/` must live on the *same filesystem* as
  `upper/` for that atomic rename trick to work, must start out empty, and
  you should never read or write to it yourself — there is nothing
  meaningful stored there for a human to inspect.
- **`merged/`** — what you see when you look through all the sheets stacked
  together: the single, combined view. Once step 3 mounts the overlay here,
  this directory transparently presents `lower/` and `upper/` combined as
  one ordinary-looking filesystem tree (`/bin`, `/etc`, `/usr`, ...). This
  is the directory that later gets `chroot`'d into (step 5) and, from that
  point forward, *becomes* the container's `/`.

This exact four-directory split — lower, upper, work, merged — isn't a
simplification invented for this lab. It's literally the mount option
signature (`lowerdir=`, `upperdir=`, `workdir=`) that Docker's `overlay2`
storage driver (and containerd's snapshotter, and most other Linux
container runtimes) constructs on every single container start.

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

**Command-by-command:**

| Command | Where it runs | Before | What it does | After / how to check |
|---|---|---|---|---|
| `curl -L -o alpine-minirootfs.tar.gz https://...` | Shell A, VM's normal (still un-namespaced) network stack | No such file in the current directory | Opens an outbound TCP/TLS connection over the VM's *real* NAT interface (`enp0s3`) — this is well before step 5/7's namespace/veth setup, so it's ordinary VM-to-internet traffic — follows any redirects (`-L`), and streams the HTTP response body to disk | `ls -la alpine-minirootfs.tar.gz` shows a ~3MB file |
| `tar -xzf alpine-minirootfs.tar.gz -C lower` | Shell A | `lower/` exists but is empty (created in step 1) | Decompresses the gzip stream, then extracts every entry from the tar archive into `lower/`, recreating each file's permissions, ownership, and symlinks exactly as recorded in the archive | `lower/` now contains a full Alpine root tree: `bin/`, `etc/`, `usr/`, `var/`, ... |
| `ls lower` | Shell A | — | Plain directory listing, purely read-only, no kernel state touched | Prints the extracted top-level entries |

**Why this step exists:** `lower/` was created empty in step 1 — it needs
actual content before an overlay mount over it means anything. We picked
Alpine specifically because it's tiny (built on `musl` libc and BusyBox
instead of glibc + GNU coreutils) and ships its own lightweight package
manager (`apk`), which makes it fast to download and fast to install
packages into later (step 8), without changing anything about how namespaces
or overlayfs work — those mechanisms are completely distro-agnostic.

**Under the hood:** this step does exactly what `docker pull` does when it
fetches an image — downloads a compressed tarball of a filesystem layer and
extracts it onto local disk into content-addressable storage. Real image
registries split a base OS into *several* tarballs (one per Dockerfile
layer) and OverlayFS supports stacking many `lowerdir`s at once
(colon-separated) to reconstruct the full image; we're using a single
tarball as one `lower/` for simplicity, but the underlying mechanism is
identical. Nothing overlay-specific happens yet — this is plain `tar`
writing files to a directory like any other extraction.

## 3. Mount the overlay

**Shell A** — the same root shell you used for steps 1–2 (the one that's
still `cd`'d into `/containerdemo` from step 1, though it doesn't actually
matter here — see the note below).

```bash
mount -t overlay overlay \
  -o lowerdir=/containerdemo/lower,upperdir=/containerdemo/upper,workdir=/containerdemo/work \
  /containerdemo/merged

ls /containerdemo/merged
```

> **Which directory do you need to be in?** None in particular — every path
> in this command (`lowerdir=`, `upperdir=`, `workdir=`, and the mountpoint
> argument) is written as an absolute path starting with `/containerdemo/...`,
> so it resolves identically no matter what your shell's current working
> directory is. This is deliberate: `mount(8)` and the overlay driver
> resolve these paths via `kern_path()` at mount time regardless of any
> notion of "current directory" — only the shell process invoking `mount`
> has a cwd, and it's irrelevant to how the kernel interprets the option
> string. You could run this exact command from `/`, `/root`, or anywhere
> else and get the same result. It must, however, run as **root** (or with
> `CAP_SYS_ADMIN`), since mounting any filesystem is a privileged operation
> — which is why this whole session is under `sudo -i`.

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `mount -t overlay overlay -o lowerdir=...,upperdir=...,workdir=... /containerdemo/merged` | Shell A, VM's default mount namespace (still un-namespaced — step 5 is what gives the container its own) | `merged/` is an empty directory; no overlay superblock exists anywhere on the system | Kernel resolves `lower/`, `upper/`, `work/`, and the mountpoint via `kern_path()`; validates `upper/`+`work/` share a filesystem; builds an in-kernel `ovl_fs` struct holding references to all three; constructs a union root inode; attaches a brand-new superblock as a `vfsmount` at `/containerdemo/merged` | `mount \| grep overlay` shows the new mount entry; `ls /containerdemo/merged` now shows Alpine's full tree even though nothing was physically copied |
| `ls /containerdemo/merged` | Shell A | — | Every lookup here is now intercepted by `ovl_lookup()`: checks `upper/` first (still empty), falls through to `lower/` for each entry | Prints Alpine's rootfs contents (`bin`, `etc`, `usr`, ...) |

`merged` now shows Alpine's filesystem. Any file the container writes lands
in `upper` — `lower` stays untouched, which is what lets you diff the
container's footprint later (step 10).

**Why this step exists:** having the four directories laid out (step 1) and
`lower/` populated (step 2) isn't enough on its own — nothing has actually
told the kernel to *combine* them yet. `mount(2)` is the syscall that
instantiates a filesystem driver (here, `overlay`) and attaches it to a
point in the directory tree. Until this runs, `merged/` is just an empty
directory; after it runs, `merged/` is backed by a live kernel object (a
superblock) that intercepts every read/write/lookup underneath it and
decides, per-file, whether to serve it from `upper/` or `lower/`.

**Under the hood — the mechanics that make `merged/` "just work":**

- **Lookup order.** When something reads a path under `merged/`, the kernel
  checks `upper/` first; if the file isn't there, it falls through to
  `lower/`. This single rule is the entirety of the "merge" — there's no
  copying or reconciliation step, just a two-tier lookup that happens on
  every access.
- **Copy-up.** The first time a *write* touches a file that currently only
  exists in `lower/`, the driver doesn't (can't — it's read-only) modify
  `lower/` in place. Instead it silently copies that file into `upper/`
  (via `work/`, atomically, as described in step 1) and then applies the
  write to the freshly-created copy in `upper/`. From then on, every access
  to that path is served from `upper/` (lookup order above). This one
  mechanism is *the entire reason* `lower/` can be shared, reused, and
  trusted to never change: writes physically cannot land there.
- **Whiteouts.** Deleting a file that only exists in `lower/` can't actually
  remove it (`lower/` is read-only from the overlay's perspective), so the
  driver instead creates a special marker in `upper/` at that path — a
  character device inode with major/minor number `0:0`, called a
  "whiteout" — which the merged view interprets as "this path is deleted,
  stop looking in `lower/`." The real file sits untouched in `lower/` the
  entire time; only the marker in `upper/` changes what's visible.
- **Opaque directories.** A similar trick — an extended attribute
  (`trusted.overlay.opaque`) set on a directory in `upper/` — tells the
  driver "don't merge this directory's contents with `lower/`'s version at
  all, `upper/`'s copy fully replaces it." This handles the case where you
  `rm -rf` an entire directory tree and recreate it from scratch.

The net effect: `merged/` behaves exactly like a complete, ordinary
filesystem to anything that reads from it — including the `chroot` in step
5 — even though physically almost nothing was duplicated. Only the parts
that actually changed ever get copied; everything else is served straight
out of the shared, untouched `lower/`.

## 4. Set up a cgroup (memory limit)

**Shell B** (host side — this cgroup will control a process we create next).

Ubuntu 22.04 mounts the unified cgroup v2 hierarchy at `/sys/fs/cgroup`.

```bash
mkdir /sys/fs/cgroup/containerdemo
echo 268435456 > /sys/fs/cgroup/containerdemo/memory.max   # 256MB cap
cat /sys/fs/cgroup/containerdemo/memory.max
```

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `mkdir /sys/fs/cgroup/containerdemo` | Shell B | No `containerdemo` cgroup exists — only the root cgroup and whatever systemd/Ubuntu already created | `mkdir(2)` on `cgroupfs` isn't a normal directory create — it's intercepted by the kernel's cgroup core, which allocates a real `cgroup` kernel object and auto-populates it with a full set of control files (`memory.max`, `memory.current`, `cgroup.procs`, `cgroup.controllers`, ...) | `ls /sys/fs/cgroup/containerdemo` shows those auto-created control files, none of which you wrote yourself |
| `echo 268435456 > /sys/fs/cgroup/containerdemo/memory.max` | Shell B | `memory.max` defaults to the string `max` (unlimited) | The `write()` syscall on this virtual file is intercepted by the kernel's memory controller, which parses the number and stores it as this cgroup's new page-charge ceiling | `cat memory.max` reads back `268435456` instead of `max` |
| `cat /sys/fs/cgroup/containerdemo/memory.max` | Shell B | — | Read-only confirmation, no state change | Prints `268435456` |

The cgroup exists but has no processes yet — you'll add the container's PID
in step 6, once it exists.

**Why this step exists:** in the next step we're about to grant a process
namespace isolation strong enough that it *feels* like its own machine — its
own PID 1, its own hostname, its own network stack. Without a resource cap,
that process (or anything it forks) could still allocate unbounded memory
and take down the entire VM, exactly the way one runaway container on a
shared Kubernetes node can starve every other pod if nobody set
`resources.limits.memory`. cgroups exist to make namespace isolation
*safe*, not just convincing.

**Under the hood:** `/sys/fs/cgroup` isn't a normal directory tree — it's
`cgroupfs`, a pseudo-filesystem the kernel exposes specifically so cgroup
management can be done with plain `mkdir`/`echo`/`cat` instead of dedicated
syscalls. Writing to `memory.max` is read by the kernel's memory controller,
which maintains a live page counter for every process charged to this
cgroup; the moment total charged memory would exceed that limit, the
kernel's OOM killer is invoked *scoped to this cgroup* — it kills a process
inside `containerdemo` to bring usage back under the cap, without touching
anything else running on the VM. This machine uses cgroup v2's *unified*
hierarchy — one single tree covering all resource controllers (memory, cpu,
pids, io, ...) — which replaced the older v1 model where every controller
had its own separate, independently mountable hierarchy and processes
could end up in inconsistent positions across them. The cgroup is
deliberately created empty: `cgroup.procs` starts with zero entries because
the process it will govern doesn't exist yet — that's fixed in step 6.

## 5. Create namespaces + chroot

**Shell A.**

`unshare` detaches the new process from the host's PID/UTS/IPC/mount/network
namespaces, then `chroot`s it into the overlay.

```bash
unshare --pid --uts --ipc --mount --net --fork \
  chroot /containerdemo/merged /bin/sh
```

You're now inside the container. Before anything else, mount a fresh
`/proc` — the one that's there right now is just an empty directory from
the Alpine tarball, not a live view into the kernel's process table:

```bash
mount -t proc proc /proc
```

> **Why this is needed even though `unshare` has a `--mount-proc` flag:**
> that flag mounts `/proc` *just before* `unshare` execs the next program —
> but the next program here is `chroot ... /bin/sh`, and the mount happens
> **before** `chroot` runs. So `--mount-proc` would mount procfs onto the
> host's real `/proc` path (isolated inside the new mount namespace, but
> still pre-chroot), and then `chroot` immediately hides it behind the new
> root. From that point on, `/proc` means `/containerdemo/merged/proc` — an
> empty directory with nothing mounted there at all. If you skip this step,
> `ps aux` will print only the header row (no processes, not even itself)
> and `top` will fail with `no process info in /proc` — there's simply no
> procfs mounted at the path either tool is reading from. Mounting it
> manually, from inside the chroot, is the only way to get a procfs that's
> both visible at the container's `/proc` *and* correctly scoped to the new
> PID namespace (which it is, because by this point the running shell is
> already inside that namespace — a `mount -t proc` always reflects the PID
> namespace of whichever process performs the mount).

Now confirm the isolation:

```bash
hostname containerdemo
hostname
ps aux                 # should show exactly one process — this shell, as PID 1
ip link                # only shows nothing or a down 'lo' — no host interfaces visible
```

Mount `/sys` too (needed later for nginx and general sanity):

```bash
mount -t sysfs sysfs /sys
```

> **Which shell, and whose `/sys`?** Still Shell A — but note this line runs
> *inside* the shell the `chroot` command above just dropped you into, not
> your original login shell (that `unshare ... chroot ... /bin/sh` call
> replaced your shell with a new one). The `/sys` here is the **container's**
> `/sys`, not the host's: because of `chroot`, this process's root is
> permanently repointed at `/containerdemo/merged`, so the kernel resolves
> the path `/sys` here as `/containerdemo/merged/sys` on the real host disk
> — but the process itself has no way to know that; it just sees `/sys`.
> And because `--mount` gave this process a *private* copy of the mount
> table before the `chroot`, this mount is only visible inside this
> namespace — the VM's real `/sys` (as seen from Shell B) is untouched.

Leave this shell open and idle here — this is your container's live shell.

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `unshare --pid --uts --ipc --mount --net --fork chroot /containerdemo/merged /bin/sh` | Shell A, still the VM's default namespaces at the moment you press enter | Shell A is an ordinary process sharing the VM's PID/UTS/IPC/mount/net namespaces; its root is `/` | `unshare(2)` forks a child (`--fork`) and detaches 5 namespace references in that child's `nsproxy` struct, replacing each with a fresh, private one; the child then calls `chroot(2)`, repointing `task_struct->fs->root` at `/containerdemo/merged`; finally it `execve()`s `/bin/sh` | You get a new shell prompt (typically `/ #`, BusyBox `ash`'s prompt). This process is PID 1 in a brand-new PID namespace, has its own hostname/IPC/mount table/network stack, and its filesystem root is `/containerdemo/merged` |
| `mount -t proc proc /proc` | Shell A, now inside the chroot | `/proc` is an empty directory (shipped in the Alpine tarball) | Mounts a fresh procfs instance at the (chrooted) `/proc`, scoped to this process's new PID namespace since that's the namespace the mounting process is in | `ps aux` and `top` now work — they read live kernel data instead of an empty directory |
| `hostname containerdemo` | Shell A | UTS namespace's hostname is whatever the VM's was at the moment of `unshare` | `sethostname(2)` writes to *this namespace's private copy only* | — |
| `hostname` | Shell A | — | Read-only | Prints `containerdemo` |
| `ps aux` | Shell A | — | Reads the freshly-mounted procfs, which is inherently filtered to processes visible in this PID namespace | Shows exactly one row: this shell, as PID 1 |
| `ip link` | Shell A | — | Reads this process's (new, empty) network namespace's interface list | Shows only `lo`, and it's `DOWN` |
| `mount -t sysfs sysfs /sys` | Shell A | `/sys` is an empty directory (from the tarball) | Mounts a fresh sysfs instance at the chrooted `/sys`, visible only inside this private mount namespace | `ls /sys` now shows real kernel/device entries instead of nothing |

**Why this step exists:** this is the single most important step in the
whole lab — it's the moment an ordinary shell process becomes, for all
practical purposes, "a container." Everything before this (overlay,
cgroup) prepared the environment; this step is what actually puts a process
*into* that environment and cuts it off from seeing the host as anything
other than the outside world.

**Under the hood — what each namespace flag actually detaches:**

`unshare` wraps the `unshare(2)` syscall (the same underlying mechanism as
`clone(2)`'s namespace flags — this is literally how `clone()` inside
`fork()`-like calls creates namespaced children). Every process has a
`task_struct` in the kernel, and every `task_struct` points to an
`nsproxy` struct bundling references to its current namespaces. Each flag
below swaps one of those references for a brand-new, empty/private one:

- **`--pid` (`CLONE_NEWPID`)** — a new PID namespace. The shell that runs
  becomes **PID 1** *inside* that namespace — that's why, once `/proc` is
  properly mounted, `ps aux` shows exactly one process (itself) — but
  the exact same process still holds an ordinary, real PID as seen from the
  VM's host namespace (visible via `/proc/<pid>/status`'s `NSpid` line,
  which lists the PID as seen from every ancestor namespace at once). PID
  namespaces nest: the host can always see every PID inside every
  namespace it's an ancestor of (with a different number), but the
  container can never see PIDs outside itself. That asymmetry — outside can
  see in, inside can't see out — is the pattern every namespace type below
  follows too. One more consequence of being "PID 1": if this shell exits,
  the kernel tears down every other process left in the namespace along
  with it (the same "reaping" behavior real init systems inside containers
  have to handle).
- **`--uts` (`CLONE_NEWUTS`)** — a private hostname/domainname. `hostname
  containerdemo` writes to this namespace's copy only; the VM's own
  hostname is untouched.
- **`--ipc` (`CLONE_NEWIPC`)** — isolated System V IPC objects (shared
  memory segments, semaphores, message queues) and POSIX message queues, so
  IPC keys used inside the container can't collide with — or be read by —
  anything on the host.
- **`--mount` (`CLONE_NEWNS`)** — a private *copy* of the mount table (note:
  a copy, not empty — every mount the host currently has is initially
  visible too). This is what lets `mount -t sysfs sysfs /sys`, and the
  `chroot` itself, happen without touching the VM's real mount table. In
  production container runtimes, this new mount namespace is also
  explicitly marked with private propagation (`MS_PRIVATE`, recursively) so
  that mount/unmount events inside the container never leak back out to the
  host's mount table — worth knowing even though `unshare` handles
  reasonable defaults for us here.
- **`--net` (`CLONE_NEWNET`)** — an entirely independent network stack:
  its own interfaces, its own routing table, its own netfilter/iptables
  rule set, its own `/proc/net`. This is why `ip link` shows essentially
  nothing usable yet — a brand-new net namespace starts *completely empty*,
  not even with loopback (`lo`) turned on. Step 7 is what wires this empty
  stack up to anything.
- **`--fork`** — `unshare` itself has to `fork()` a child to actually enter
  the new PID namespace cleanly (a process can't become PID-1-of-a-new-
  namespace without being created *after* the namespace exists).

Note there's no `--mount-proc` flag on this command, even though `unshare`
offers one — see the callout above for why it doesn't help in a
chroot-based setup like this one, and why `/proc` gets mounted by hand
instead, after the `chroot`.

**Then `chroot /containerdemo/merged /bin/sh`** changes which directory the
kernel treats as `/` for this process's path lookups (technically, it
updates `task_struct->fs->root`). This is worth understanding as
*deliberately weaker* than the namespaces above: `chroot` predates
namespaces by decades and, used alone, is not a real security boundary — a
sufficiently privileged process that retains an open file descriptor to the
old root (opened before the `chroot` call) can escape it entirely by
`fchdir()`-ing back through that descriptor and `chroot`-ing again from
there. Production container runtimes use `pivot_root(2)` instead, which
fully swaps the entire mount tree's root — old root included — out of
reach, then unmounts it, closing that escape hatch. This lab uses plain
`chroot` because it's simpler to reason about while learning the
overlay/namespace mechanics; it's a genuine simplification, not an
oversight, and worth remembering if you ever read real runtime source code.

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

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `ps aux \| grep "[c]hroot /containerdemo/merged"` | Shell B | — | Reads the **host's own** `/proc` (Shell B was never namespaced, so it sees the VM's real, full process table). Because PID namespaces nest, every process inside the container's PID namespace is *also* visible here — just under its real, host-assigned PID number instead of its in-container number (1) | Prints one line: the container shell's real, host-visible PID |
| `echo <PID> > /sys/fs/cgroup/containerdemo/cgroup.procs` | Shell B | `cgroup.procs` is empty (step 4 created the cgroup with nothing in it) | The `write()` on this control file is intercepted by cgroup core: validates the PID refers to a live process, then atomically updates that task's `css_set` pointer — the internal structure tying it to cgroup membership across every controller at once | — |
| `cat /sys/fs/cgroup/containerdemo/cgroup.procs` | Shell B | — | Read-only confirmation | Shows the PID now listed as a member |

**Why this step exists:** the cgroup was created empty in step 4 because the
process it needed to govern didn't exist yet. Now it does (step 5 created
it), so this step is simply the missing link — connecting the resource cap
that already exists to the process that's now running.

**Under the hood:** from the moment the PID is written into `cgroup.procs`,
every page of memory this process (or anything it later forks) allocates
gets charged against `containerdemo`'s `memory.max`, and any child process
automatically inherits its parent's cgroup membership — you don't need to
repeat this step for processes the container spawns later (like `nginx`,
in step 8). Worth noting for realism: this two-step "create process, then
move it into a cgroup" approach has a small race window where the process
briefly runs *before* being resource-constrained. Modern container
runtimes like `runc` close that gap using the `clone3(2)` syscall's
`CLONE_INTO_CGROUP` flag (Linux 5.7+), which places a process into a
target cgroup atomically as part of its creation — no window at all. We're
using the older, two-step approach here because it's easier to observe and
reason about by hand.

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

**Baseline, before running the block above** — worth checking these in Shell
B first so you can see the "before" state for yourself: `ip link show` only
lists the VM's real interfaces (`lo`, `enp0s3`, `enp0s8` — no `veth-*` yet);
`ip route show` has no route to `10.200.1.0/24`; `cat
/proc/sys/net/ipv4/ip_forward` reads `0`; `iptables -t nat -L POSTROUTING`
and `iptables -L FORWARD` are both empty. In Shell A, `ip link show` shows
only a `DOWN` loopback — nothing else exists.

**Command-by-command** — every row runs in **Shell B** (the host/VM's own
network namespace); none of these touch the container's namespace directly
except the one row called out below:

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `ip link add veth-host type veth peer name veth-ctr` | Shell B, host root netns | No `veth-*` devices exist | The veth driver allocates **two** linked devices in one atomic call — a permanently paired unit from creation, not two devices connected later. Both land in Shell B's current namespace, both `DOWN`, neither has an IP. | `ip link show` in Shell B lists both `veth-host` and `veth-ctr`, state `DOWN`. |
| `ip link set veth-ctr netns $CPID` | **Issued from Shell B, but acts on the container's namespace** | `veth-ctr` sits in the host's netns | The kernel looks up the netns owned by PID `$CPID` (via `/proc/$CPID/ns/net`) and **reparents** `veth-ctr` into it — a real move, not a copy. | `ip link show` in Shell B again → `veth-ctr` is **gone** from this namespace entirely. It now only exists inside the container (visible from Shell A, or via `nsenter -t $CPID -n ip link` from Shell B). `veth-host` stays behind, untouched. |
| `ip addr add 10.200.1.1/24 dev veth-host` | Shell B | `veth-host` has no address | Assigns `10.200.1.1/24`. Side effect: the kernel auto-inserts a connected route, `10.200.1.0/24 dev veth-host ... src 10.200.1.1` — you never typed this route; every IP assignment implicitly creates one for its subnet. | `ip route show` in Shell B now has that `10.200.1.0/24` entry. |
| `ip link set veth-host up` | Shell B | `veth-host` administratively `DOWN` | Sets the `IFF_UP` flag on this end only. A veth pair only reports full "carrier up" once *both* ends are administratively up. | `ip link show veth-host` → `UP`, but may still show `NO-CARRIER`/`LOWER_LAYERDOWN` until Shell A brings `veth-ctr` up next. |
| `sysctl -w net.ipv4.ip_forward=1` | Shell B | `0` — forwarding disabled | Writes `1` to `/proc/sys/net/ipv4/ip_forward`. This sysctl is itself **per-network-namespace** — it only enables forwarding inside Shell B's (the VM's) namespace; the container has its own independent copy, irrelevant here since the container never routes traffic for anyone else. | `cat /proc/sys/net/ipv4/ip_forward` → `1`. |
| `iptables -t nat -A POSTROUTING -o "$UPLINK" -j MASQUERADE` | Shell B | `nat`/`POSTROUTING` chain empty | Appends a rule to netfilter's `nat` table at the `POSTROUTING` hook. Doesn't touch existing traffic — a standing rule that fires on future packets leaving via `$UPLINK`. | `iptables -t nat -L POSTROUTING -n` shows the `MASQUERADE` rule. |
| `iptables -A FORWARD -i veth-host -o "$UPLINK" -j ACCEPT` | Shell B | `filter`/`FORWARD` chain empty | Appends a permit rule to the default (`filter`) table's `FORWARD` chain for container→internet traffic. | `iptables -L FORWARD -n` shows this rule. |
| `iptables -A FORWARD -i "$UPLINK" -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT` | Shell B | (same chain, second rule) | Appends the return-path rule, scoped to `RELATED,ESTABLISHED` using the conntrack table the MASQUERADE rule populates — only replies to connections the *container itself* started are let back in, not arbitrary unsolicited inbound traffic. | `iptables -L FORWARD -n` shows two rules now. |

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

**Command-by-command — every row runs in Shell A, inside the container's
namespace:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `ip link set lo up` | Shell A | `lo` exists (every net namespace gets one automatically) but is `DOWN` | Sets `IFF_UP` on the container's *own private* loopback — a fresh net namespace's `lo` starts down, unlike a normal machine's | `ip link show lo` → `UP` |
| `ip addr add 10.200.1.2/24 dev veth-ctr` | Shell A | `veth-ctr` exists (moved in by Shell B in the previous block) but has no address | Assigns the address inside this namespace; also auto-creates the `10.200.1.0/24` connected route, but in the *container's own* private routing table, separate from Shell B's | `ip addr show veth-ctr` shows it |
| `ip link set veth-ctr up` | Shell A | `veth-ctr` `DOWN` on this end | Brings this end up too — now *both* ends of the pair are administratively up, so carrier state flips to fully `UP` on both sides simultaneously | `ip link show veth-ctr` → `UP`; back in Shell B, `veth-host` also flips from `NO-CARRIER` to full `UP`/`LOWER_UP` |
| `ip route add default via 10.200.1.1` | Shell A | Container's routing table only has the connected `10.200.1.0/24` route | Adds a `0.0.0.0/0` route via the host side of the veth link, inside the container's own private routing table | `ip route show` (inside the container) shows `default via 10.200.1.1` |
| `echo nameserver 8.8.8.8 > /etc/resolv.conf` | Shell A | Whatever Alpine's tarball shipped for `/etc/resolv.conf` (typically empty/absent) | A plain file write — but because this is under the chroot, it's served through the overlay: since the file only existed (or didn't) in `lower/`, this write triggers copy-up into `upper/` (step 3's mechanism, in action for the first time) | Container's resolver now queries `8.8.8.8` for DNS |
| `ping -c 2 10.200.1.1` | Shell A | — | Sends two ICMP echo requests out `veth-ctr`, exercising the entire path just built (veth pair + both ends' addressing) | Two replies confirm host↔container connectivity works |

**Why this step exists:** `--net` in step 5 gave the container a network
stack so private it's actually *disconnected* — it can't talk to anything,
including the host it's running on. That's correct and expected (it's real
isolation, not a bug), but it also means the container can't install
`nginx` yet. This step builds the smallest possible bridge between two
otherwise-unreachable networks: a virtual cable plus enough routing/NAT
rules to make it feel like an ordinary machine with internet access.

**Under the hood — the full packet path, piece by piece:**

- **The veth pair.** `veth-host`/`veth-ctr` are created together as a
  linked pair — think of it as a virtual Ethernet cable with a plug on each
  end. A packet written to one end is handed directly to the other end's
  receive path by the kernel's veth driver. There is no real wire, no
  switch, no physical layer involved at all — it's a synchronous, in-kernel
  handoff between two network devices that happen to live in different
  namespaces.
- **`MASQUERADE`** is a netfilter rule installed into the `nat` table's
  `POSTROUTING` hook (one of several fixed points in the kernel's packet
  processing pipeline that netfilter lets you attach rules to). As a packet
  from the container leaves via `$UPLINK`, this rule rewrites its source
  address from the private `10.200.1.2` to the VM's own address — and,
  using the kernel's connection-tracking table (`conntrack`), automatically
  remembers that translation so the *reply* packet gets un-rewritten and
  routed back to the right container on the way in. This is the exact
  mechanism behind consumer home routers sharing one public IP among many
  devices, and it's the same mechanism Docker's default bridge network uses
  to give containers outbound internet access.
- **The `FORWARD` chain rules** are the actual permission gate: enabling
  `ip_forward` makes forwarding *possible*, but netfilter's `FORWARD` chain
  decides whether any given packet crossing between `veth-host` and
  `$UPLINK` is actually allowed through. Without these two rules, forwarding
  would be enabled at the kernel level but every packet would still be
  dropped.
- **Note on tooling:** on Ubuntu 22.04, the `iptables` command you're
  running is actually a compatibility shim (`iptables-nft`) translating
  these rules into the kernel's newer `nftables` subsystem — the concepts
  above (tables, chains, hooks, conntrack) are identical either way, only
  the underlying rule representation differs.

This entire step — veth pair + bridge/NAT — is precisely what Docker's
default bridge network does for every container, and what CNI plugins do
under the hood for every pod in Kubernetes; we're just doing it by hand
instead of letting a daemon script it for us.

## 8. Install and run nginx inside the container

**Shell A.**

```bash
apk update
apk add nginx curl
nginx
curl -i 127.0.0.1
```

> **Expect a `404 Not Found`, not a welcome page — and that's correct.**
> Unlike Debian/Ubuntu's `nginx` package, Alpine's ships a default vhost
> that deliberately returns `404` for *every* path:
> `/etc/nginx/http.d/default.conf` contains `location / { return 404; }`,
> with the comment "This is a default site configuration which will simply
> return 404, preventing chance access to any other virtualhost" — a
> security default against Host-header/vhost hijacking, not a missing file
> (there's a perfectly good `index.html` sitting unused at
> `/var/lib/nginx/html/index.html`). A `404` response here is actually
> *better* proof the stack works than a static welcome page would be: it
> means the request crossed the container's private loopback, reached
> nginx's listening socket, and nginx's own server-block logic executed and
> returned a deliberate answer — real application behavior, not just a
> file read.

> **If you run `nginx` a second time** (e.g. re-running this block after
> already completing it once), it will fail with
> `bind() to 0.0.0.0:80 failed (98: Address in use)` — this is expected,
> not an error to fix. It means the *first* nginx master process is still
> running and still holding port 80 in this namespace; the second `nginx`
> invocation simply can't take a port that's already bound. `curl` will
> keep working fine against the original instance. Check with `ps aux |
> grep nginx`, or `pkill nginx` first if you actually want a clean restart.

**Command-by-command:**

| Command | Where it runs | Before | What it does | After / how to check |
|---|---|---|---|---|
| `apk update` | Shell A | No local package index cached inside the container | Uses the container's own DNS + routing (both just built in step 7) to fetch Alpine's repo index over HTTP(S), through the veth+NAT path, into local cache | `apk` now knows what package versions are available |
| `apk add nginx curl` | Shell A | `nginx` binary/config absent from the rootfs; `curl` isn't in the Alpine minirootfs either — only BusyBox's built-in applets (`wget`, etc.) ship by default, unlike the VM host, which has real `curl` | Downloads both `.apk` packages and extracts their files into the container's filesystem — every write goes through the overlay, landing in `upper/` via copy-up — then runs any post-install scripts each package ships | `/usr/sbin/nginx`, `/etc/nginx/`, `/usr/bin/curl`, etc. now exist, and are visible in `upper/` (see step 10) |
| `nginx` | Shell A | Nothing bound to port 80 in this network namespace *(first run only — see note below)* | Starts the nginx master process, which forks worker processes (they automatically inherit this container's PID/mount/net namespaces — no need to `unshare` again), and binds a listening socket on `0.0.0.0:80` scoped to this namespace's own socket table | `ps aux` shows nginx's master + worker processes; a listening socket exists on `:80` |
| `curl -i 127.0.0.1` | Shell A | — | Connects via the container's *own* private loopback (from step 7, separate from the host's `lo`), delivered straight to nginx's listening socket; nginx's default server block matches the request and executes its configured `return 404` | Prints `HTTP/1.1 404 Not Found` plus an HTML body with `nginx` in the footer — this is a real response *from* nginx, not a connection failure |

**Why this step exists:** everything so far has been infrastructure — a
filesystem, isolation, a resource cap, a network path. This step runs an
actual, real-world workload inside all of it, proving the container isn't
just isolated in theory but is a genuinely usable environment: it can
install software over the network and serve traffic like any ordinary Linux
box.

**Under the hood:** `apk update`/`apk add nginx curl` reaching Alpine's
package mirror is the first real, end-to-end proof that step 7's plumbing
works — it needs *both* a route out through the veth+NAT path *and* working
DNS resolution, all happening inside what is, from the kernel's point of
view, a completely independent network stack with its own sockets and
routing table. Once `nginx` binds to port 80 and `curl -i 127.0.0.1`
returns an HTTP response (404 or otherwise), note that this loopback is the
container's *own* private loopback interface — a second nginx bound to
port 80 on the VM's real host network namespace could coexist without any
conflict at all, because "port 80" is scoped per-network-namespace, not
global to the machine. This is exactly why dozens of containers on one
Docker host can each think they own port 80.

## 9. Reach the container from the host

**Shell B.**

```bash
curl -i 10.200.1.2
```

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `curl -i 10.200.1.2` | Shell B (the VM host already has `curl` — it was installed by the `Vagrantfile`'s provisioner, unlike the Alpine container in step 8) | Shell B already has a route to `10.200.1.0/24` via `veth-host` (auto-added in step 7) | The VM's routing table sends the connection out `veth-host`; the veth driver hands it straight to `veth-ctr` on the container side — no NAT/uplink involvement at all, since host and container are directly, mutually routable over the veth link; the container's own TCP/IP stack completes the handshake and delivers the connection to nginx's listening socket | Prints the exact same `HTTP/1.1 404 Not Found` response as step 8's `curl -i 127.0.0.1` — same nginx process, same default vhost logic, just reached over the veth link instead of the container's own loopback |

If this returns the same `404` response as step 8, the full path —
namespaces, veth, routing, NAT/forwarding rules — is working end to end.
(A matching `404` is the success condition here, not a failure — see the
callout in step 8 for why Alpine's `nginx` responds this way by default.)

**Why this step exists:** step 8 proved the container can reach *out*. This
step proves the reverse direction — that something *outside* the container
can reach *in* — closing the loop and confirming the whole network path is
genuinely bidirectional, not an accident of NAT.

**Under the hood:** trace the packet in the opposite direction from step 7.
That MASQUERADE/FORWARD machinery is only needed for the container reaching
*outward* to the wider internet, where it needs a translated source
address; here, host and container are directly, mutually routable over the
veth link, so nothing needs to be rewritten. Contrast this with real Docker
port publishing (`-p 8080:80`): because Docker containers usually live
behind a bridge on a private subnet the host doesn't automatically route
to, exposing a port externally requires an explicit DNAT rule on the host
redirecting traffic — here we skip that only because we gave the container
a directly-routable IP by hand.

## 10. Inspect the overlay diff

**Shell B.**

Everything nginx's install and startup touched lives in `upper/`, while
`lower/` (the pristine Alpine base) is untouched:

```bash
find /containerdemo/upper -maxdepth 3
diff <(cd /containerdemo/lower && find . | sort) \
     <(cd /containerdemo/merged && find . | sort) | head -50
```

**Command-by-command:**

| Command | Where it runs | Before | What it does | After / how to check |
|---|---|---|---|---|
| `find /containerdemo/upper -maxdepth 3` | Shell B | — | Lists everything that physically exists in `upper/` — every entry is either a copy-up from `lower/` (step 3's mechanism) or a wholly new file `nginx`/`apk` created, or a whiteout marker for something deleted | Prints paths like `etc/nginx/...`, `var/log/nginx/...`, `run/nginx.pid` |
| `diff <(...lower...) <(...merged...)` | Shell B | — | Compares a sorted listing of the pristine `lower/` tree against the union `merged/` tree (as seen through the overlay) | Shows exactly the paths that are new or changed — nginx's full on-disk footprint |

This is the same mechanism a real container runtime uses to build image
layers and compute `docker diff`-style output.

**Why this step exists:** step 3 explained copy-up in the abstract; this
step makes it concrete by showing you the actual evidence it left behind.
It's also the step that ties this whole exercise back to how container
*images* get built in the first place, not just how containers get run.

**Under the hood:** every file `apk add nginx` installed, every config file
it touched, and everything `nginx` itself wrote while running (PID files,
logs, temp files) triggered a copy-up into `upper/` at the moment it was
first written — `lower/` remains byte-for-byte the same Alpine tarball you
extracted in step 2, verifiable directly by this diff. Walking `upper/`'s
contents is *exactly* the algorithm real container tooling uses: `docker
diff <container>` reads a container's upper (writable) layer the same way;
`docker commit` tars up that same upper layer and registers it as a new,
immutable image layer, which becomes tomorrow's `lower/` for someone else's
container. You are looking, by hand, at precisely the artifact a real
container build pipeline would package and push to a registry.

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

Or, for a fully clean slate, just discard the whole VM:

```bash
vagrant destroy -f
```

and re-run `vagrant up` to start the lab over from scratch.

**Command-by-command:**

| Command | Where it runs | Before | What it does in the kernel | After / how to check |
|---|---|---|---|---|
| `exit` | Shell A | The chrooted `/bin/sh` (PID 1 of its own PID namespace) is running | Terminates that process. Because `--fork` made it a *child* of the original `unshare` command, control returns to that parent — your original login shell, back in the VM's normal, un-namespaced world. Since this was the last (and only) process in all five namespaces `unshare` created, their reference counts drop to zero and the kernel frees the `pid_namespace`, `uts_namespace`, `ipc_namespace`, mount namespace, and `net_namespace` objects — anything mounted only inside that private mount namespace (the container's `/proc`, `/sys`) is torn down automatically as part of that | Your terminal shows the *original* root shell prompt again (not the `/ #` one) — you're back in Shell A's un-namespaced starting point |
| `iptables -t nat -D POSTROUTING -o "$UPLINK" -j MASQUERADE` | Shell B | Rule present (from step 7) | Removes the matching rule from the `nat` table's `POSTROUTING` chain (matched by full rule spec, not position) | `iptables -t nat -L POSTROUTING -n` no longer shows it |
| `iptables -D FORWARD ...` ×2 | Shell B | Both `FORWARD` rules present | Removes each matching rule from the `filter` table's `FORWARD` chain | `iptables -L FORWARD -n` is empty again |
| `ip link delete veth-host` | Shell B | `veth-host` exists (its peer `veth-ctr` lived inside the container's now-defunct netns) | Destroys the `veth-host` device; since a veth pair shares one underlying kernel object between its two ends, this also removes the peer, even though it was last seen inside a namespace that no longer exists | `ip link show` no longer lists `veth-host` |
| `umount /containerdemo/merged` | Shell B | Overlay superblock active, `merged/` shows the union view | Releases the overlay's in-kernel superblock; `lower/` and `upper/` themselves are untouched, ordinary directories on disk | `ls /containerdemo/merged` shows an empty directory again |
| `rmdir /sys/fs/cgroup/containerdemo` | Shell B | Cgroup is empty of processes (the container's shell already exited) | Removes the cgroup kernel object — cgroup v2 refuses this if any process is still a member | `ls /sys/fs/cgroup` no longer lists `containerdemo` |
| `vagrant destroy -f` | **Not Shell A or B — run this on your real host machine's terminal**, outside the VM entirely, from the directory containing the `Vagrantfile` | VM is running | Powers off and deletes the entire VirtualBox VM and its virtual disk | `vagrant up` is required again for a completely fresh VM |

**Why this step exists:** namespaces, cgroups, veth interfaces, and mounts
are all live kernel state — none of it disappears automatically just
because you're done looking at it. Skipping cleanup on a real (non-disposable)
host would leave a phantom cgroup, dangling veth interface, and stale
iptables rules behind indefinitely.

**Under the hood:** namespaces are reference-counted kernel objects, not
files you explicitly delete. Each namespace is actually exposed as an inode
on a special pseudo-filesystem called `nsfs`, reachable via
`/proc/<pid>/ns/{pid,net,mnt,uts,ipc}`. A namespace is only actually freed
once its reference count hits zero — meaning the last process inside it has
exited *and* nothing else (an open file descriptor, a bind mount onto one of
those `nsfs` inodes) still references it. That's why simply exiting the
`unshare`d shell is enough here to tear down all five namespaces it
created — but it's also worth knowing this is exactly how tools like `ip
netns add` deliberately *keep* a namespace alive with no process running in
it at all: by bind-mounting its `nsfs` inode to a persistent path under
`/var/run/netns/`, holding the reference count above zero indefinitely.
