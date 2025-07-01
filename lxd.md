# lxd

see official documentation [here](https://documentation.ubuntu.com/lxd/stable-5.21/).  
lxd is a container and vm management daemon built on top of lxc (the low-level linux containers library).  it presents a unified rest api and cli (lxc) for managing full-system containers (and, since v4.0+, virtual machines).  at its core, lxd is a privileged host daemon (lxd) that uses the liblxc library to create and manage instances.  the daemon can be accessed locally via a unix socket (requiring group membership) or remotely over https/tls with client certificates.  unlike docker-style app containers, lxd containers share the host kernel and emulate entire linux os environments, making them more like lightweight vms.  as the image below illustrates, lxd unifies management of containers (which share the host kernel) and traditional virtual machines (which run their own guest os) under one platform.

```
+-------------------------------------------------------+
|                   lxd host (linux)                    |
|                                                       |
|  +----------------------+   +---------------------+   |
|  |  system container a  |   | system container b  |   |
|  | (shares host kernel) |   | (shares host kernel)|   |
|  +----------------------+   +---------------------+   |
|                                                       |
|  +---------------------+                              |
|  |  virtual machine c  |                              |
|  | (runs guest kernel) |                              |
|  +---------------------+                              |
|                                                       |
|  lxd daemon manages both types uniformly via api      |
+-------------------------------------------------------+

```

## lxd architecture

lxd’s architecture centers on the daemon (lxd), which implements all functionality behind a rest api.  the lxc command-line client is just a thin wrapper that sends http requests to this api.  the lxd daemon handles cluster membership, storage pool management, network configuration, and container/vm lifecycle.  internally, lxd stores all state (instances, profiles, images, networks, projects, etc.) in a local dqlite database (sqlite with built-in raft replication).  in a standalone server, this is just a sqlite file on disk; in a cluster, dqlite and the raft algorithm keep the state in sync among nodes.  components include:

* daemon and apis: the lxd daemon runs as root, publishing a rest api (at /1.0 endpoints) that reflects all objects (instances, images, networks, storage pools, etc.). all lxc cli operations and external tools (go or python clients) ultimately use this api.  interactive operations (console i/o, migration streams, file push/pull, event streams) use upgraded websockets negotiated over the rest connection.  administrators can also use the lxd command (for debug or init) and manage the daemon via systemctl or snap interfaces.
* container/instance model: each container or vm is an “instance” object with configuration (limits, security, devices, networks), state (running/stopped/snapshotted), and its rootfs.  containers and vms use different runtimes: containers run directly via lxc (fork/execing into the user namespace), whereas vms run via virtqemud (qemu) and use a firmware agent (lxd-agent or cloud-init) inside the guest.  instances can have profiles which supply common config and devices (e.g. an eth0 network device by default).  the expanded configuration is computed by applying profiles in order, then instance overrides.  instances are typically persistent (long-lived), but may be flagged ephemeral for single-use testing.
* storage: lxd abstracts storage via storage pools which map to backends (zfs, btrfs, lvm, ceph, or plain dir). each instance gets its rootfs on a volume (block device or subvolume) in the pool.  lxd handles snapshotting and cloning efficiently using the backend’s copy-on-write features. for example, a zfs pool clones from an image or snapshot with minimal i/o, and ceph rbd pools create thin-cloned images.  custom storage volumes (block or filesystem) can also be attached to instances.  (see storage backends below for details.)
* networking: lxd can create and manage networks (bridges, vlans, sdn overlays).  by default it configures a linux bridge (e.g. lxdbr0) with nat and dhcp for containers.  more advanced modes include macvlan (direct host interface subinterfaces), ovn-managed sdn overlays across hosts, and fan networks (an lxd-specific flat-overlay addressing).  lxd automatically configures network devices inside containers and sets up routing, firewalling, dns as needed.
* security: lxd uses linux kernel security primitives extensively to isolate instances.  by default containers run unprivileged in a user namespace (mapped to non-root uids on the host). additional hardening includes seccomp syscall filtering, apparmor profiles to restrict file/socket access, dropped linux capabilities, and cgroup resource limits.  administrators can opt for “privileged” containers (no user-namespace) if absolutely needed, but lxd’s safe defaults favor maximal isolation.  lxd also supports external authentication; it originally used a trust-password/key system for tls client certs, and now integrates with canonical’s candid authentication gateway for enterprise ldap/sso login.

## container lifecycle management

lxd manages containers’ full lifecycle via its api/cli. key operations and behaviors include:

* instance launch/creation: lxc launch <image> <name> fetches an image (from local cache or remote server), creates a new root filesystem (volume) by cloning or copying, applies profiles, and starts the container or vm.  the image serves as a pristine template (clean distro rootfs).  lxd stores the base-image fingerprint in volatile.base_image.  containers can also be created from raw disks or from scratch.  lxd supports ephemeral containers (auto-deleted on stop) and normal persistent ones.
* snapshots and cloning: lxd lets you take snapshots of running or stopped instances via lxc snapshot.  a snapshot is logically an immutable clone: it records the container’s state and (if using criu) even its memory/cpu state.  internally, a snapshot is a new volume cloned from the current one (leveraging zfs/btrfs/ceph cow).  snapshots can be renamed, restored, or deleted.  restoring a snapshot makes the container revert to the saved state (including killed processes rolled back with criu).  lxd supports stateful snapshots for live instances (checkpoint/restore).  a container can be “published” to an image (making a new image from it), which effectively snapshots and adds to the image store.
* live and offline migration: lxd can migrate instances between hosts.  by default, migrations are done in pull mode: the destination lxd pulls data from the source lxd.  for containers, if the storage backend supports efficient sends (zfs, btrfs, ceph rbd) lxd will use those; otherwise it falls back to rsync over a network.  for live migration, lxd establishes three synchronized websocket streams: control, filesystem, and (if stateful) criu dump stream.  these streams allow iterative pre-copy and a final stop-the-world dump.  on the destination side, volumes are received and mounted, and criu restores state.  apis like instance/migrate negotiate the best protocol, and report progress events.  vms use standard qemu migration over libvirt/qemu spiffe.
* deletion: lxc delete stops and removes an instance, deleting its rootfs volume, configuration, and records from the db.  by default, all snapshots of the instance are also removed (unless --force is used).  if the storage backend has no thin clone capability (e.g. dir), the rootfs is just deleted; otherwise cow clones are freed.  any attached custom volumes remain by default (unless --purge is specified to remove them).  lxd cleans up any network namespace or virtual network interfaces that were allocated.
* hooks/events: lxd does not use user-defined init scripts in containers by design (unlike raw lxc hooks).  instead, it provides lifecycle events (container-started, instance-stopped, etc.) that clients can listen to via the events api.  there are some internal “template” hooks: e.g. volatile.apply_template can trigger container templating on next start, but arbitrary host-side hooks are discouraged.  instead, users usually implement pre/post actions by watching lxd events or by wrapping lxc calls in scripts.

## storage backends and configuration

lxd supports multiple storage drivers. each driver defines how instance rootfs volumes and image stores are managed:

* directory (dir): plain filesystem directory storage. each container is just a folder on the host filesystem. this is simple but has no cow snapshots: cloning or snapshotting must copy files. useful for small test setups or backing lxd’s own db.
* zfs: uses zfs pools. lxd can auto-create a zfs zpool (usually a loop device by default) or use an existing one. each container and snapshot is a zfs filesystem. snapshots and clones are very efficient (metadata only). supports quotas, compression, and snapshots can be rolled back atomically. zfs-backed images can use zfs send/receive for rapid transfer.
* btrfs: uses btrfs subvolumes. each instance is a subvolume/snapshot. btrfs supports cow snapshots but has fewer enterprise features than zfs. note: quota management on btrfs is more manual; enabling quotas is often recommended for container volumes.
* lvm: uses lvm logical volumes (thinly-provisioned by default). lxd can create a thin-pool in an lvm vg and carve lv for each instance. supports cow clones if thin-provisioning is used. if not using a thinpool, lxd falls back to plain lv creation. supports resizing volumes. in clusters, lvm volumes are typically host-local (no shared pg).
* ceph rbd: uses ceph block (rbd) pools. each volume is a ceph rbd image. supports cow clones and snapshots natively. lxd must be configured with ceph credentials. this backend allows distributed storage across hosts. cephfs (ceph filesystem) can also be used for filesystem volumes, but is mainly for custom non-rootfs volumes.
* other: lxd also supports powerflex, pure, and ssh-backed pools, but the above are most common.

each storage pool is configured on init or via lxc storage create.  images are stored once per pool and cloned to instances.  lxd automatically reuses storage driver features: e.g., it will call zfs send/receive when moving snapshots to speed up data transfer.  custom volumes (block or filesystem) can be created and attached to instances for additional disks.

## networking modes and internals

lxd supports several network types for instance connectivity:

* bridged networking (default): the common “lxdbr0” linux bridge connects instances to each other and (by default) to the outside world via nat.  lxd will run a dhcp/dns server on the bridge unless told otherwise.  containers get ips on the bridge subnet and can reach the internet through the host.  the bridge can optionally have vlan tags and custom routes.  (lxd also supports using open vswitch for bridges if desired, though linux bridges are typical.)
* macvlan/ipvlan: assigns each container an interface on a parent physical nic.  with macvlan, each container gets its own mac address on the parent network. containers cannot talk to the host but appear as separate machines on the lan.  with ipvlan, ipv4/6 subnetting can be used directly. these are useful when you want containers on the same l2 network as the host without nat or bridging.
* ovn (open virtual network): lxd can integrate with an ovn/ovs deployment for a software-defined overlay network.  you create an ovn network in lxd (with type=ovn) and connect instances to it; lxd handles the necessary logical switches/routers in ovn.  internally, an ovn network is represented by a bridge (lxdovnn) on each host that connects to ovn’s integration bridge (br-int), as shown below.  the example diagram illustrates a 3-node ovn cluster: each lxd host has a logical switch for the network, connected to a distributed logical router. the lxd-managed lxdovnx bridges ensure containers join the overlay (vxlan) network across hosts【100†】.

```
+------------------+     +-------------------+     +-------------------+
|  lxd node 1      |     |  lxd node 2       |     |  lxd node 3       |
|                  |     |                   |     |                   |
| +--------------+ |     | +---------------+ |     | +---------------+ |
| |lxdovn2 bridge| |     | | lxdovn2 bridge| |     | | lxdovn2 bridge| |
| +------+-------+ |     | +------+--------+ |     | +------+--------+ |
|        |         |     |        |          |     |        |          |
+--------|---------+     +--------|----------+     +--------|----------+
         |                        |                         |
         |                        |                         |
         +-----------+------------+-------------------------+
                     |
            +--------------------+
            | ovn logical switch |
            +---------+----------+
                      |
              +-------+--------+
              | ovn logical    |
              | distributed    |
              | router         |
              +-------+--------+
                      |
            +--------------------+
            | ovn integration br |
            | (br-int spans all  |
            | nodes in overlay)  |
            +--------------------+
```

* fan (flat address network): a special lxd overlay mode for cross-host flat addressing.  when a bridge’s bridge.mode=fan, lxd sets up a vxlan (or ipip) overlay using a flat /8 address space (default 240.0.0.0/8). containers on different hosts get ips from this overlay subnet and can route to each other automatically via the underlay.  the key parameters (fan.overlay_subnet, fan.underlay_subnet, fan.type) configure the overlay network. this is a lightweight alternative to kubernetes-like flannel for simple flat networking.

each network has many tunables (dhcp range, mtu, ipv6 support, dns zones, etc). in practice, most setups use the default bridge for simplicity, with macvlan for direct lan access, ovn for multi-host l2 domains, and fan for cloud-like overlay networking.  lxd also supports sr-iov and physical passthrough interfaces as “external networks”.

## clustering and high availability

lxd can run in cluster mode: multiple lxd servers (nodes) share a single logical cluster with one dqlite-based database.  a typical cluster has 3 or 5 nodes (odd for quorum) in at least two failure domains. key points:

* distributed db (dqlite/raft): the cluster’s entire state (instances, networks, images, etc.) lives in a distributed sqlite (dqlite) database replicated via the raft consensus algorithm.  one node is bootstrap (initial leader), others join as replicas.  in a 3-node cluster all nodes hold a copy of the db; for larger clusters lxd replicates to a majority subset (to limit load).
* roles and failover: nodes that hold db replicas can be voters or standby.  the leader coordinates commits; if it fails, raft elects a new leader among voters.  if a voting node goes offline, a standby is auto-promoted to voter, maintaining quorum.  thus lxd remains available as long as a majority of members are up.  non-replica nodes still accept management commands (they proxy reads to leader).
* joining a cluster: after bootstrapping one server, additional nodes join with lxc cluster add.  the new node gets copies of the storage pools’ metadata (for local pools) and a replica of the db.  storage pools themselves remain local to each host (e.g. zfs pools aren’t shared across machines) unless using a networked backend (like ceph) that all can see.
* uniform management: once clustered, any node can handle requests. the lxc client can be pointed at any node and will get a consistent view.  instance operations (create, move, etc.) go through the leader which coordinates the db and migration, but operations are transparent to users.  the cluster can migrate instances between nodes with shared pools or external storage.  lxd handles fencing: if a node is partitioned, raft ensures split-brain is avoided.  heartbeats and timeouts govern failover, and administrators can manually evict or re-add nodes if needed.
* high availability: with 3+ nodes, lxd provides basic ha for container management (not automatic service restart).  if the leader node dies, the cluster stays up on the new leader with no data loss.  however, container workloads themselves still run on individual nodes; lxd does not auto-restart containers on another node (no live ha), though you could script evacuation (e.g. migrate containers off a failing node).  the cluster’s fault-tolerance comes from the replicated database and the fact that images, profiles, and configs are shared.

## security model

lxd employs multiple layers of linux security to isolate instances:

* namespaces: containers run in separate namespaces (pid, net, mount, etc.).  crucially, user namespaces are enabled by default: container uid 0 is mapped to a non-root id on the host. this means processes inside an lxd container do not have full root rights on the host (they’re “unprivileged”).  privilege escalation (like mknod or mounting) is blocked or filtered unless specifically allowed.  lxd can run containers as privileged (no mapping) if the user opts in, but this is not the default.
* apparmor: lxd loads a confined apparmor profile for each container to restrict what it can do.  apparmor policies limit file access (especially cross-container or host-protected areas), ptrace, and network namespace interactions. for example, containers cannot see host’s process trees due to apparmor enforcement of namespaces.
* seccomp: a kernel seccomp filter is applied to containers, dropping dangerous syscalls (like keyctl, iopl, or kernel module ops).  the default profile (security.syscalls.deny_compat*) blocks 32-bit emulation syscalls and many legacy calls. users can fine-tune security.syscalls.allow/deny lists if needed.
* capabilities and cgroups: containers start with a reduced set of linux capabilities by default (no raw i/o, no module loading, no bpf, etc.).  all mount points and devices are mediated by the kernel: lxd uses cgroups to limit resource usage (cpu shares, memory, io) for dos protection.  by default containers cannot break out of their cgroup or increase privileges.
* acl and id mapping: lxd carefully sets up /etc/subuid and /etc/subgid ranges for root’s sub-ids (usually 100000–165536).  each unprivileged container gets a slice of ids, and all filesystems are shifted accordingly (via newuidmap).  this ensures even if a container had uid 0, it only matches a non-privileged host id.  changing this mapping mid-life is complex, so lxd generally fixes the mapping at container creation.
* authentication (candid): for access control to lxd’s api, lxd by default uses a trusted tls certificate mechanism.  clients add themselves as trusted via an ssh-like trust password.  in enterprise setups, lxd can delegate auth to candid (canonical’s web-based auth gateway).  with candid, administrators configure group-based access so that users log in via existing ldap/sso, and lxd checks the candid server for authorization. this allows centralized user/group control and easy revoke via identity providers.

## image management

lxd is image-based. containers and vms are launched from images: prebuilt filesystems or vm disks for various linux distributions. lxd ships with defaults for remote image servers and automated updates:

* remote servers: by default lxd is pre-configured with “ubuntu:”, “ubuntu-daily:”, and “images:” remotes.  ubuntu: provides canonical’s official cloud images (lts releases, snaps, etc.), ubuntu-daily: provides daily builds, and images: is a community mirror (images.linuxcontainers.org) with dozens of distro images (fedora, alpine, arch, etc.).
* caching and auto-updates: when you launch or pull an image from a remote, lxd caches it locally (in its “images” storage).  cached images expire after 10 days by default (as a safety) and can be purged with lxc image delete if needed.  lxd will also auto-refresh images on remotes by default: it periodically checks for newer versions of aliases and fingerprints and updates them so that e.g. “ubuntu:20.04” refers to the latest 20.04 release unless you opt out.
* aliases and properties: each image has a sha256 fingerprint and may have aliases (like ubuntu:20.04).  you can add your own local aliases for images.  images can also carry metadata (release series, architecture) and signatures (especially in commercial settings).  the lxc image commands allow listing, importing (image import or lxc image copy --alias), and exporting images (to files).
* publishing containers: you can turn a container or vm into an image via lxc publish. this snapshots the instance and adds it to the local image store (optionally uploading to a remote). this is how you build custom images. lxd updates the cache metadata so your lxc launch <alias> pulls the updated image.

## rest api structure

everything in lxd is exposed via its json rest api (versioned under /1.0). key points:

* http/json api: resources include /1.0/instances/…, /1.0/images/…, /1.0/networks/…, /1.0/storage-pools/…, etc.  most standard crud operations are mapped to rest verbs (get list or item, post to create, put/patch to modify config, delete to remove).  responses include etag headers for consistency checks.  there are also “command” endpoints (e.g. /exec, /console) which upgrade to websockets.
* events: a special endpoint /events streams json events (lxc, lxd, cluster events) over a long-lived http connection (with keep-alive).  clients can subscribe to watch lifecycle events, migrations, etc.
* authentication: locally, lxd relies on unix socket permissions (members of lxd group).  remotely, each lxd server has a tls server cert; clients present their user cert (added via lxc config trust) to authenticate.  the api returns http status codes and json error messages.
* extensions: the api is extensible. lxd advertises supported extensions in the get /1.0 response (like “storage\_zfs\_remove\_snapshots”, “gpu\_devices”, etc.) so clients know what features the server supports. the core is kept backward compatible (v1.0 was stabilized in lxd 2.0, and 1.x remains supported).
* usage: developers can use the api directly via curl or http libraries. in practice, most use the pylxd python library or go’s lxc/client package to script lxd. many configuration tasks can also be done via lxc cli (lxc config set, lxc network attach, etc.), which simply calls the api under the hood.

## tooling and automation

* lxc cli: the primary user-facing tool is the lxc command (note: not lxd).  it provides subcommands for almost all operations: lxc launch, lxc list, lxc config, lxc exec, lxc file, lxc snapshot, lxc image, etc.  it supports --project, --profile, and remote selection, mirroring the api paths.
* lxd cli: there is also an lxd command for daemon management (init, cluster join, version, migrate, etc.). this is mainly for administrators; everyday container ops use lxc.
* sdks: python’s pylxd library (github.com/canonical/pylxd) and go’s lxd/client (pkg.go.dev/lxc/lxd/client) provide programmatic clients.  they handle auth and json marshalling.  many operators write go or python tools to manage lxd clusters.
* scripting: since the api is restful, you can script lxd with any language (even bash + curl).  however, the official tools and sdks are more convenient.  for example, continuous integration pipelines often call lxc exec to run tests inside containers, or use lxc profile to configure networking for automated jobs.
* integration: lxd can integrate with openstack (nova-lxd driver) and juju/maas for cloud operations.  it also has a metrics api (prometheus endpoint) and can dump db with lxd sql for external tools.

## performance tuning and optimization

when deploying lxd at scale, consider the following:

* benchmarking: lxd includes a built-in lxd-benchmark tool. this can spin up many containers/vms and measure create times.  by running benchmarks under different settings (e.g. with/without snapshots on storage, varying kernel params, etc.), you can identify bottlenecks.
* metrics & monitoring: lxd exports per-instance metrics (cpu, ram, disk i/o, network) suitable for prometheus.  you should set up a monitoring stack (prometheus + grafana) to track resource use and spot issues.  watch for hot-spots like high disk usage or orphaned processes in containers.
* kernel and sysctl: the linux defaults may limit container count.  tune /proc/sys/kernel/* (max pid, pid reservations), file descriptor limits, and network table sizes as documented in the lxd production guide.  for example, raising fs.inotify.max_user_instances may be needed if many containers run services watching files.  the guide lists common errors (“too many open files”, neighbor table overflow) and sysctl fixes.
* network tuning: if instances do heavy networking, increase nic queue depths.  for example, raise /proc/sys/net/core/rps_sock_flow_entries or txqueuelen on bridges as explained in lxd docs.  if using bridges, ensure bridge-nf-call-iptables=0 if you don’t need host-to-container netfilter, for performance.
* storage performance: choose a backend suited to your workload.  zfs and btrfs incur some cpu overhead for cow metadata.  lvm or ceph may provide better raw i/o throughput (especially with fast disks/ssd).  ensure that the pool’s disk cache or block size matches your use case.  for heavy disk iops, consider a dedicated ssd pool.
* huge pages & cpu pinning: for vms or high-performance containers, you can enable ksm (ksm.sharing enabled) or hugepages, and pin vcpus or vnics via lxd cpu/numa rules to optimize latency.  lxd allows setting limits.cpu and limits.cpu.priority for real-time scheduling.
* container distribution: spread instances across nodes to balance load. lxd’s clustering does not auto-schedule; plan ahead. use lxc move to rebalance if one host is overloaded.

## operational practices

* backups: always back up critical data.  for lxd, a full backup is taken by saving the lxd data directory (/var/lib/lxd or for snap installs /var/snap/lxd/common/lxd) along with /etc/subuid and /etc/subgid.  this directory contains the db, config, and local storage.  note: external volumes (lvm vgs, zfs pools, ceph osds) must be backed up separately.  to restore, stop lxd, replace the directory, restore external storage, and restart lxd.  for finer-grained backups, use lxc export <instance> to create tarballs of instances (including snapshots), or use a second “backup” lxd server: periodically copy instances (lxc copy --refresh) and custom volumes to it.  remember that snapshots alone are not a safe backup (they live on the same pool).
* live migration: use lxc move --mode=live for live migration of containers (with criu) or --stateless/--stateful to include running state.  ensure source and target use compatible storage backends (or skip --optimized if not).  lxd’s migration will automatically use criu if the container is stopped (lxc move --stateful) or attempt live criu iterations if running.
* scaling: to scale to many containers, use multiple lxd hosts in a cluster.  even without clustering, you can script distributing containers by “projects” or separate lxd servers.  the official guidance is to keep a few dozen vms/containers per host for manageability, scaling out to clusters of tens of nodes for large deployments.
* monitoring: besides prometheus, use lxc list and lxc info to check on instance states.  the lxd database can reveal resource usage (lxd sql local "select * from instances").  monitor the lxd daemon health via lxd.monas or systemd.  in clusters, watch raft status with lxc cluster list and lxc cluster info <name>.
* high availability ops: in a cluster, if a node fails you can lxc cluster remove <node> and later re-add it after fixing.  for planned maintenance, migrate instances off a node first.  use lxc cluster recover if the cluster loses quorum (guide in lxd docs).

## security and troubleshooting

* logs and debugging:  check instance logs with lxc info <name> --show-log for lxc errors.  for container console logs, use lxc console <name> --show-log.  these reveal startup errors (e.g. missing /dev/null) and systemd failures.  host-side lxd logs go to syslog or journalctl -u snap.lxd.daemon (if using snap) or /var/log/lxd/lxd.log.  you can also run lxc monitor --type=logging --pretty to watch real-time lxd daemon debug output (even on local socket).  the lxd.buginfo tool (snap) bundles host information for support tickets.
* common issues: resource limits (ulimits, sysctl) often cause failures (oomkills, “too many open files”, namespace conflicts).  enabling more debug (--debug on lxc commands) helps.  for networking issues, verify bridge-nf-call settings and firewall rules.  storage errors usually mean pools are full or missing.  in clusters, ensure clock sync (ntp) and verify raft health (lxc cluster list).
* recovery: the docs include a [cluster recovery](https://documentation.ubuntu.com/lxd/stable/cluster_recovery/) procedure if a majority of nodes fail.  for individual instances, you can try lxc recover on stuck containers.  for worst-case, exports of images and lxd sql local .dump of the db allow reconstructing state.
* security practices: regularly update lxd (snap tracks). scan instance images for cves. use apparmor/selinux policies if applicable. limit ssh keys to root in containers if you allow ssh. keep the host kernel updated to benefit from isolation fixes.