RustFS Installation
===================

Installs [RustFS](https://rustfs.com) - a high-performance, S3-compatible,
distributed object storage server - on Ubuntu 24.04 virtual machines, managed
by `systemd`. Supports **air-gapped** environments, applies OS kernel tuning
and enables `io_uring` for better I/O throughput.

References:
- https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html
- https://docs.rustfs.com/upgrade-scale/availability-and-resiliency.html

Requirements
------------

- Ubuntu 24.04 (kernel 5.x/6.x). `io_uring` enablement targets kernel >= 6.6.
- `ansible.posix` collection (for the `sysctl` module).
- Each node should have its data disks **formatted XFS and mounted in JBOD
  mode** at the paths listed in `rustfs_datadirs` (the role creates the
  directories but does not format or mount disks).
- A minimum of **4 nodes per storage pool** for a production multi-node
  multi-disk deployment, with identical disk specs across the pool.
- Sequential, resolvable hostnames (DNS or `/etc/hosts`) matching the
  `{x...y}` expansion in `rustfs_volumes`, and time sync across all nodes.

Role Variables
--------------

See [defaults/main.yml](defaults/main.yml) for the full list. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_version` | `latest` | Release used to build the download URL. |
| `rustfs_download_url` | dl.rustfs.com zip | Binary archive URL (online deploys). |
| `enable_download_binary` | `true` | `false` for air-gapped (pre-stage binary). |
| `rustfs_download_local_dir` | `/tmp` | Control-node staging dir for the binary. |
| `rustfs_bin_path` | `/usr/local/bin/rustfs` | Install path on targets. |
| `rustfs_user` / `rustfs_group` | `root` | Service identity (set a dedicated user to harden). |
| `rustfs_address` | `:9000` | S3 API listen address. |
| `rustfs_console_enable` / `rustfs_console_address` | `true` / `:9001` | Web console. |
| `rustfs_access_key` / `rustfs_secret_key` | `rustfsadmin` | Root credentials - **change these**. |
| `rustfs_volumes` | 4x4 expansion | `RUSTFS_VOLUMES`, identical on every node. |
| `rustfs_datadirs` | `/data/rustfs0..3` | Data directories to create. |
| `rustfs_log_dir` / `rustfs_log_level` | `/var/logs/rustfs` / `error` | Logging. |
| `rustfs_tune_kernel` | `true` | Apply sysctl performance tuning. |
| `rustfs_enable_io_uring` | `true` | Set `kernel.io_uring_disabled=0`. |
| `rustfs_configure_iptables` | `false` | Open API/console ports to cluster peers. |

### Kernel tuning applied

Written to `/etc/sysctl.d/90-rustfs.conf`:

```
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_slow_start_after_idle = 0
vm.swappiness = 10
net.core.somaxconn = 32768
kernel.io_uring_disabled = 0   # when rustfs_enable_io_uring: true
```

Air-gapped deployment
---------------------

1. On a machine with internet access, download and extract the binary:
   ```
   wget https://dl.rustfs.com/artifacts/rustfs/release/rustfs-linux-x86_64-musl-latest.zip
   unzip rustfs-linux-x86_64-musl-latest.zip   # produces ./rustfs
   ```
2. Copy the extracted `rustfs` binary to the Ansible **control node** at
   `{{ rustfs_download_local_dir }}/rustfs` (default `/tmp/rustfs`).
3. Run the role with `enable_download_binary: false`. The role copies the
   pre-staged binary to every target host - no internet access required on
   the targets or control node.

Example Playbook
----------------

```yaml
- hosts: rustfs-cluster
  roles:
    - role: storage/rustfs
      vars:
        rustfs_access_key: "rustfsadmin"
        rustfs_secret_key: "a-long-random-secret"
        rustfs_volumes:
          - "http://node{1...4}:9000/data/rustfs{0...3}"
        rustfs_datadirs:
          - /data/rustfs0
          - /data/rustfs1
          - /data/rustfs2
          - /data/rustfs3
```

A working multi-node example lives in [tests/install_rustfs.yml](tests/install_rustfs.yml).

> **Note:** RustFS forbids rolling restarts - all nodes must restart together.
> Run the play against the whole cluster at once (no `serial`) so handlers fire
> across every node in the same run. Use `--limit` only when *adding* a new
> storage pool.

License
-------

BSD

Author Information
------------------

Created for the vhkt automation repository.
