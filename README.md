RustFS Installation
===================

Installs [RustFS](https://rustfs.com) 1.0.0 - a high-performance, S3-compatible,
distributed object storage server - on Ubuntu 24.04 virtual machines, managed
by `systemd`. Supports **air-gapped** environments, applies OS kernel tuning,
installs the required standard tools and enables the opt-in `io_uring` read
backend for better I/O throughput.

References:
- https://docs.rustfs.com/en/installation/linux/prerequisites-and-service
- https://docs.rustfs.com/en/installation/requirement/disk-preparation
- https://docs.rustfs.com/en/installation/requirement/checklists/software-checklists
- https://github.com/rustfs/rustfs/blob/main/docs/operations/io-uring-initialization.md
- https://github.com/rustfs/rustfs/blob/main/crates/obs/README.md

Requirements
------------

- Ubuntu 24.04 (Linux 5.x/6.x). The `io_uring` read backend requires kernel
  5.10+ to be effective.
- `ansible.posix` collection (for the `sysctl` module).
- Standard tools `bash`, `glibc` (`libc6`) and `openssl` are installed by the
  role (see `rustfs_system_packages`); `ca-certificates` is included for TLS.
- Each node should have its data disks **formatted XFS and mounted in JBOD
  mode** at the paths listed in `rustfs_datadirs`. The role creates the
  directories but does **not** format or mount disks - see
  [Disk preparation](#disk-preparation) below.
- A minimum of **4 nodes per storage pool** for a production multi-node
  multi-disk deployment, with identical disk specs across the pool.
- Sequential, resolvable hostnames (DNS or `/etc/hosts`) matching the
  `{x...y}` expansion in `rustfs_volumes`, and time sync across all nodes.

Disk preparation
----------------

Disks are formatted and mounted **outside** this role (formatting is
destructive and host-specific). Prepare each dedicated data disk as an
independent XFS file system with a stable label/mount point on every node
before running the role. See the upstream guide:
https://docs.rustfs.com/en/installation/requirement/disk-preparation

Quick version:

```
# 1. Identify the dedicated data disks (do NOT touch the OS disk).
sudo lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS

# 2. Format XFS with a label (repeat per disk: RUSTFS1, RUSTFS2, ...).
sudo mkfs.xfs -f -i size=512 -n ftype=1 -L RUSTFS0 /dev/sdb

# 3. Create the mount point and a persistent fstab entry.
sudo mkdir -p /data/rustfs0
echo 'LABEL=RUSTFS0 /data/rustfs0 xfs defaults,noatime,nodiratime 0 0' | sudo tee -a /etc/fstab

# 4. Mount and verify (XFS + expected free space).
sudo mount -a
df -hT /data/rustfs0
sudo xfs_info /data/rustfs0
```

Use one mount point and one `/etc/fstab` entry per disk, matching the paths in
`rustfs_datadirs`. Keep labels and mount paths consistent across all nodes.

Role Variables
--------------

See [defaults/main.yml](defaults/main.yml) for the full list. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_version` | `1.0.0` | Release used to build the download URL. |
| `rustfs_download_url` | GitHub release zip | Binary archive URL (online deploys). |
| `enable_download_binary` | `true` | `false` for air-gapped (pre-stage binary). |
| `rustfs_download_local_dir` | `/tmp` | Control-node staging dir for the binary. |
| `rustfs_bin_path` | `/usr/local/bin/rustfs` | Install path on targets. |
| `rustfs_install_packages` | `true` | Install `rustfs_system_packages`. |
| `rustfs_system_packages` | `bash`, `libc6`, `openssl`, `ca-certificates` | Standard tools. |
| `rustfs_user` / `rustfs_group` | `rustfs` | Service identity (set `root` to match upstream). |
| `rustfs_address` | `:9000` | S3 API listen address. |
| `rustfs_console_enable` / `rustfs_console_address` | `true` / `:9001` | Web console. |
| `rustfs_access_key` / `rustfs_secret_key` | `rustfsadmin` | Root credentials - **change these**. |
| `rustfs_volumes` | 4x4 expansion | `RUSTFS_VOLUMES`, identical on every node. |
| `rustfs_datadirs` | `/data/rustfs0..3` | Data directories to create. |
| `rustfs_log_dir` / `rustfs_log_level` | `/var/log/rustfs` / `error` | Logging. |
| `rustfs_tune_kernel` | `true` | Apply sysctl performance tuning. |
| `rustfs_enable_io_uring` | `true` | Set `RUSTFS_IO_URING_READ_ENABLE`. |
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
```

> **io_uring:** since RustFS 1.0.0 `io_uring` is an opt-in **read** backend,
> disabled by default. The role enables it with
> `RUSTFS_IO_URING_READ_ENABLE=true` when `rustfs_enable_io_uring: true` - no
> kernel-wide `kernel.io_uring_disabled` sysctl is needed (that stale setting
> was removed). Optional tuning is available via `rustfs_io_uring_shards`,
> `rustfs_io_uring_max_driver_threads` and the shared read-budget variables.

### Log cleanup

Rolling-file logging and automatic cleanup are handled by `rustfs-obs`
([reference](https://github.com/rustfs/rustfs/blob/main/crates/obs/README.md)).
The role renders these into the environment file:

| Variable | Default | Purpose |
|----------|---------|---------|
| `rustfs_log_max_total_size_bytes` | `2147483648` | Hard cap on total log dir size (2 GiB). |
| `rustfs_log_max_single_file_size_bytes` | `0` | Per-file cap; `0` = unlimited. |
| `rustfs_log_keep_files` | `30` | Rolling files to keep. |
| `rustfs_log_rotation_time` | `hourly` | `minutely` / `hourly` / `daily`. |
| `rustfs_log_compress_old_files` | `true` | Compress before deleting. |
| `rustfs_log_compression_algorithm` | `zstd` | `zstd` or `gzip`. |
| `rustfs_log_compressed_file_retention_days` | `30` | Delete archives older than N days; `0` = forever. |
| `rustfs_log_cleanup_interval_seconds` | `1800` | Cleanup task interval. |
| `rustfs_log_parallel_compress` | `true` | Work-stealing parallel compression. |
| `rustfs_log_parallel_workers` | `6` | Cleaner worker threads. |
| `rustfs_log_zstd_compression_level` | `8` | zstd level 1-21. |
| `rustfs_log_dry_run` | `false` | Report deletions without removing files. |

Air-gapped deployment
---------------------

1. On a machine with internet access, download and extract the binary:
   ```
   wget https://github.com/rustfs/rustfs/releases/download/1.0.0/rustfs-linux-x86_64-musl-v1.0.0.zip
   unzip rustfs-linux-x86_64-musl-v1.0.0.zip   # produces ./rustfs
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
