RustFS Installation
===================

Installs [RustFS](https://rustfs.com) 1.0.5 - a high-performance, S3-compatible,
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
- https://docs.rustfs.com/en/administration/protocols/ftps
- https://docs.rustfs.com/en/integration/tls-configured
- [CHANGELOG.md](CHANGELOG.md) - release notes and configuration changes

Requirements
------------

- Ubuntu 24.04 LTS (the role targets 24.04; upstream supports Ubuntu 20.04+
  and RHEL 8/9). Linux 5.10 or later is recommended; the optional `io_uring`
  read backend requires it.
- `ansible.posix` collection (for the `sysctl` module) and `community.crypto`
  (for certificate generation and inspection) - install both with
  `ansible-galaxy collection install -r requirements.yml`.
- Standard tools `bash`, `glibc` (`libc6`) and `openssl` are installed by the
  role (see `rustfs_system_packages`); `ca-certificates` is included so the
  server can verify TLS peers against the system trust store (RustFS 1.0.4
  refreshed certificate verification and trust roots). `openssl` is no longer
  required on the control node - the self-signed FTPS pair is generated with
  `community.crypto`.
- Each node should have its data disks **formatted XFS and mounted in JBOD
  mode** at the paths listed in `rustfs_datadirs`. The role creates the
  directories but does **not** format or mount disks - see
  [Disk preparation](#disk-preparation) below.
- A minimum of **4 nodes per storage pool** for a production multi-node
  multi-disk deployment, with identical disk specs across the pool.
- Sequential, resolvable hostnames (DNS or `/etc/hosts`) matching the
  `{x...y}` expansion in `rustfs_volumes`, and time sync across all nodes.

Pre-installation checklist
--------------------------

The role implements the host-side items from the upstream
[pre-installation checklists](https://docs.rustfs.com/en/installation/requirement/checklists)
([software](https://docs.rustfs.com/en/installation/requirement/checklists/software-checklists),
[network](https://docs.rustfs.com/en/installation/requirement/checklists/network-checklists),
[hardware](https://docs.rustfs.com/en/installation/requirement/checklists/hardware-checklists),
[security](https://docs.rustfs.com/en/installation/requirement/checklists/security-checklists)).
Some items are deliberately outside the role because they are host- or
organization-specific.

**Handled by the role**

- Installs the standard tools and root certificates (`bash`, `glibc`/`libc6`,
  `openssl`, `ca-certificates`), the binary under `/usr/local/bin` with mode
  `0755`, and the same version on every node.
- Verifies the release `SHA256SUMS` before installing and verifies TLS on the
  download itself.
- Creates a dedicated, no-login service user/group and grants it access to the
  data and log directories.
- Applies the four sysctls from the network checklist and installs a hardened
  `systemd` unit (`User=`, `EnvironmentFile=`, `ExecStart=`, resource limits).
- Keeps CORS same-origin by default and FTPS certificates private (`0700`
  directory, `0600` key).

**Operator responsibilities**

- **OS/kernel:** install Ubuntu 20.04+/RHEL 8/9 with Linux 5.10+, and keep the
  OS, OpenSSL and RustFS patched.
- **Interfering services:** disable file indexing and auditing services such
  as `mlocate`, `auditd` and antivirus so they do not compete with RustFS I/O.
- **Time sync and DNS:** configure chrony/`systemd-timesyncd` on every node
  (drift under 1s in production) and make all `rustfs_volumes` hostnames
  resolvable.
- **Disks:** XFS on dedicated JBOD disks, no NFS, no hardware RAID; mount by
  filesystem UUID and make the mount paths match `rustfs_datadirs`.
- **Network:** homogeneous nodes, a separate storage network (leaf-spine,
  redundant links, 25/100 GbE for production), jumbo frames only when the
  whole path supports them, and firewall openings for TCP 9000/9001 (plus 443
  when a TLS reverse proxy terminates HTTPS).
- **Memory and capacity:** at least 2 GB for a test node; the hardware guide's
  entry profile is 64 GB per node (the Linux prerequisites page recommends
  128 GB for production). Size bandwidth and drives from the hardware guide.
- **SELinux/AppArmor:** allow RustFS access to its binary, config, data and log
  paths, or disable the policy if appropriate for your environment.
- **TLS:** provide a CA-issued server certificate when exposing HTTPS; the
  role does not manage the S3 TLS listener.
- **Observability, backups and rollback:** configure Prometheus/Grafana,
  audit-log export, configuration backups and a tested rollback plan.
- **Data protection:** enable bucket versioning and Object Lock where
  regulatory retention (WORM) applies.
- **Production validation:** run the upstream 72-hour stress test (failover,
  network partition, 120% burst writes) before go-live.

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
#    Upstream recommends mounting by filesystem UUID (stable across reboots
#    and duplicate labels); LABEL=RUSTFS0 works too.
sudo mkdir -p /data/rustfs0
UUID=$(sudo blkid -s UUID -o value /dev/sdb)
echo "UUID=$UUID /data/rustfs0 xfs defaults,noatime,nodiratime 0 0" | sudo tee -a /etc/fstab

# 4. Mount and verify (XFS + expected free space).
sudo mount -a
df -hT /data/rustfs0
sudo xfs_info /data/rustfs0
```

Use one mount point and one `/etc/fstab` entry per disk, matching the paths in
`rustfs_datadirs`. Keep labels/UUIDs and mount paths consistent across all
nodes.

Role behaviour
--------------

**Files managed by the role (replaced, not modified).** The role owns the
following paths outright; any local edit is overwritten on the next run (the
`template` tasks keep a `backup: true` copy):

| Path | Owner / mode |
|------|--------------|
| `{{ rustfs_envfile }}` (`/etc/default/rustfs`) | `rustfs:rustfs`, `0640` |
| `/etc/systemd/system/{{ rustfs_service_name }}.service` | `root:root`, `0644` |
| `{{ rustfs_sysctl_file }}` (`/etc/sysctl.d/90-rustfs.conf`) | `root:root` |
| `{{ rustfs_ftps_certs_dir }}` (`/etc/rustfs/ftps-certs`) | `rustfs:rustfs`, `0700` |
| `{{ rustfs_log_dir }}`, `{{ rustfs_working_directory }}`, `{{ rustfs_datadirs }}` | `rustfs:rustfs` |

**Idempotent: yes.** A second run against an unchanged host reports
`changed=0`. Service restarts are driven by handlers, so the unit is only
restarted when the binary, the unit file or the environment file changes.
Note that RustFS requires the whole cluster to restart together, so a single
node's change restarts every node in the run.

**Check mode: supported.** `ansible-playbook --check` does not modify the
hosts. Two documented exceptions:

- On a host where the binary has not been staged yet, the binary copy is
  skipped instead of diffed (the role asserts on a staged binary in normal
  mode).
- With `rustfs_ftps_self_signed: true`, generation is skipped while the private
  key is absent (check mode cannot create the certificate directory), and the
  certificate assertions are skipped for that one case.

**Rollback.** The role has no automatic rollback. To undo a deployment:

1. `systemctl disable --now {{ rustfs_service_name }}`.
2. Remove `/etc/systemd/system/{{ rustfs_service_name }}.service`,
   `{{ rustfs_envfile }}` and `{{ rustfs_sysctl_file }}`, then
   `systemctl daemon-reload` and `sysctl --system`.
3. Remove the data directories **only** after confirming that no node still
   participates in the same storage pool - removing them on a live cluster
   destroys the pool.
4. To go back one version, re-run the role with the previous `rustfs_version`.
   Rolling back across a format change is not supported by upstream.

Role Variables
--------------

See [defaults/main.yml](defaults/main.yml) for the full list. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_version` | `1.0.5` | Release used to build the download URL. |
| `rustfs_artifact_name` | `rustfs-linux-x86_64-gnu-v<version>.zip` | Release artifact (swap for arm64/musl). |
| `rustfs_download_url` | GitHub release zip | Binary archive URL (online deploys). |
| `rustfs_verify_checksum` / `rustfs_checksums_url` | `true` / release `SHA256SUMS` | Verify the archive before unpacking. |
| `rustfs_download_validate_certs` | `true` | Verify TLS on control-node downloads. |
| `rustfs_enable_download_binary` | `true` | `false` for air-gapped (pre-stage binary). |
| `rustfs_download_local_dir` | `~/.cache/rustfs` | Control-node staging dir (created with mode `0700`). |
| `rustfs_bin_path` | `/usr/local/bin/rustfs` | Install path on targets. |
| `rustfs_install_packages` | `true` | Install `rustfs_system_packages`. |
| `rustfs_system_packages` | `bash`, `ca-certificates`, `libc6`, `openssl` | Standard tools. |
| `rustfs_user` / `rustfs_group` | `rustfs` | Service identity (set `root` to match upstream). |
| `rustfs_working_directory` | `/opt/rustfs` | systemd `WorkingDirectory` (created by the role). |
| `rustfs_address` | `:9000` | S3 API listen address. |
| `rustfs_region` | `""` | `RUSTFS_REGION`; used for bucket location validation. |
| `rustfs_internode_http_protocol` | `auto` | Internode data plane: `auto` (TLS ALPN) or `http1`. |
| `rustfs_console_enable` / `rustfs_console_address` | `true` / `:9001` | Web console. |
| `rustfs_console_prefix` | `""` | Console URL path prefix for reverse-proxy subpaths. |
| `rustfs_console_oidc_session_duration` | `null` | OIDC console credential lifetime in seconds (1-604800). |
| `rustfs_veeam_sosapi_enable` | `true` | Veeam SOSAPI virtual system objects (1.0.5). |
| `rustfs_startup_peer_readmission_enable` | `true` | Hold startup readiness until peers re-admit the node (1.0.5). |
| `rustfs_startup_readiness_max_wait_seconds` | `null` | Peer re-admission budget (upstream default 120s). |
| `rustfs_scanner_walkers_per_disk` | `null` | Scanner walkers per local disk (1-16). |
| `rustfs_scanner_disk_io_budget_iops` / `..._bytes_per_sec` | `null` | Per-disk scanner I/O ceilings (`auto`/`off`/number). |
| `rustfs_scanner_recovery_backoff_cap_seconds` | `null` | Cap on scanner recovery retry backoff. |
| `rustfs_cors_allowed_origins` | `""` | S3 endpoint CORS allow-list (same-origin only when empty). |
| `rustfs_console_cors_allowed_origins` | `""` | Console CORS allow-list (same-origin only when empty). |
| `rustfs_ftp_enable` / `rustfs_ftp_address` | `false` / `127.0.0.1:8021` | Plain FTP listener (testing only; loopback by default). |
| `rustfs_ftp_passive_ports` / `rustfs_ftp_external_ip` | `40000-50000` / `""` | FTP passive range and NAT-advertised address. |
| `rustfs_ftps_enable` / `rustfs_ftps_address` | `false` / `0.0.0.0:8022` | Explicit FTPS (FTPES) listener. |
| `rustfs_ftps_certs_dir` | `/etc/rustfs/ftps-certs` | Directory holding `rustfs_cert.pem` and `rustfs_key.pem`. |
| `rustfs_ftps_self_signed` | `false` | Generate a test-only self-signed certificate. |
| `rustfs_ftps_self_signed_*` | see defaults | CN, SANs, key size, validity and renewal window. |
| `rustfs_ftps_passive_ports` / `rustfs_ftps_external_ip` | `40000-50000` / `""` | FTPS passive range and NAT-advertised address. |
| `rustfs_access_key` / `rustfs_secret_key` | **required** | Root credentials - no default is shipped (see note below). |
| `rustfs_rpc_secret` | `""` | Internode RPC secret; derived from the key pair when empty. |
| `rustfs_volumes` | 4x4 expansion | `RUSTFS_VOLUMES`, identical on every node. |
| `rustfs_datadirs` | `/data/rustfs0..3` | Data directories to create. |
| `rustfs_log_dir` / `rustfs_log_level` | `/var/log/rustfs` / `error` | Logging. |
| `rustfs_tune_kernel` | `true` | Apply sysctl performance tuning. |
| `rustfs_enable_io_uring` | `true` | Set `RUSTFS_IO_URING_READ_ENABLE`. |
| `rustfs_io_uring_read_chunk_bytes` | `null` | Per-operation io_uring read cap (4 KiB-128 MiB). |
| `rustfs_bind_interface` / `rustfs_bind_address` | `eth0` / derived | Cluster NIC and its IPv4 address; override the address for a VIP. |
| `rustfs_configure_iptables` | `false` | Open API/console ports to cluster peers. |

Advanced 1.0.4/1.0.5 opt-ins that this role does not model directly - audit
denial rate limits, the notify/audit target queue byte budget, MRF lifecycle
V2 and internode HTTP/2 over TLS ALPN - can be passed through
`rustfs_env_extra`. See [CHANGELOG.md](CHANGELOG.md) for the details and
preconditions.

### Kernel tuning applied

Written to `/etc/sysctl.d/90-rustfs.conf`:

```
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_slow_start_after_idle = 0
```

These are exactly the four settings from the upstream
[network checklist](https://docs.rustfs.com/en/installation/requirement/checklists/network-checklists).
The role does not apply undocumented extras (such as `vm.swappiness` or
`net.core.somaxconn`); add them through `rustfs_sysctl_settings` if your
environment needs them.

> **io_uring:** since RustFS 1.0.x `io_uring` is an opt-in **read** backend,
> disabled by default. The role enables it with
> `RUSTFS_IO_URING_READ_ENABLE=true` when `rustfs_enable_io_uring: true` - no
> kernel-wide `kernel.io_uring_disabled` sysctl is needed (that stale setting
> was removed). Optional tuning is available via `rustfs_io_uring_shards`,
> `rustfs_io_uring_max_driver_threads`, the shared read-budget variables and
> `rustfs_io_uring_read_chunk_bytes` (added in 1.0.1).

### Log cleanup

Rolling-file logging and automatic cleanup are handled by `rustfs-obs`
([reference](https://github.com/rustfs/rustfs/blob/main/crates/obs/README.md)).
The systemd unit sends stdout/stderr to the journal; `rustfs-obs` writes the
rotated files configured below to `rustfs_log_dir`. The role renders these into
the environment file:

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

FTP(S) gateway
--------------

RustFS can expose buckets over plain FTP or explicit FTPS (FTPES). Both
listeners are disabled by default. Authenticate with a RustFS access key as the
username and its secret key as the password; IAM policies are enforced per
operation. `RMD` deletes the bucket and all its objects recursively, and
rename/append operations are not supported. The standard RustFS release binary
includes the `ftps` compile-time feature, so no custom build is required.

> **Plain FTP is unencrypted.** It sends credentials and data in clear text and
> is for loopback or isolated test networks only. The role defaults
> `rustfs_ftp_address` to `127.0.0.1:8021` (upstream defaults to
> `0.0.0.0:8021`); do not expose it to untrusted networks.

For a loopback-only test listener:

```yaml
rustfs_ftp_enable: true
rustfs_ftp_address: "127.0.0.1:8021"
rustfs_ftp_passive_ports: "40000-40010"
```

### FTPS

Enable explicit FTPS with a CA-issued certificate pair:

```yaml
rustfs_ftps_enable: true
rustfs_ftps_address: "0.0.0.0:8022"
rustfs_ftps_passive_ports: "40000-40100"
# Only when clients connect through NAT or a load balancer:
# rustfs_ftps_external_ip: "storage.example.com"

# Holds rustfs_cert.pem + rustfs_key.pem (CA-issued or generated). When
# rustfs_ftps_self_signed is true the role also keeps the CSR here.
rustfs_ftps_certs_dir: /etc/rustfs/ftps-certs
rustfs_ftps_self_signed: false
```

The role creates the directory with mode `0700`, sets the private key to `0600`
and the certificate to `0644`, and verifies that the certificate is not expired
and that its public key matches the private key. Clients must use
**explicit FTP over TLS** (FTPES); implicit FTPS is not supported. Certificate
SANs must cover every hostname or IP address clients use.

For a **test-only** self-signed certificate:

```yaml
rustfs_ftps_enable: true
rustfs_ftps_self_signed: true
rustfs_ftps_self_signed_ip_sans:
  - 10.0.20.21
```

The role generates a 4096-bit RSA, SHA-256 certificate valid for 365 days with
`CA:false`, `serverAuth` EKU, key identifiers and SANs. The SAN list always
includes the common name, inventory/host/FQDN names, `localhost`, `127.0.0.1`
and the default IPv4 address; add any other names or addresses clients use. The
certificate is regenerated when it expires within
`rustfs_ftps_self_signed_renew_days` (default 30). Delete the files to force
regeneration after changing SAN settings.

Self-signed certificates are for testing and isolated internal use only. They
are not publicly trusted, and clients must explicitly trust the certificate. In
production, install a CA-issued pair and keep certificate verification enabled.
The private key has no passphrase because the systemd service must read it
unattended; the `0700` directory and `0600` key permissions are the protection.

Passive mode needs inbound TCP on the FTPS control port (8022 by default) and
on every port in `rustfs_ftps_passive_ports`; set `rustfs_ftps_external_ip`
when clients connect through NAT. The role's optional cluster firewall only
opens the S3/console ports to peers and does not cover FTP clients.

Test the listener and certificate with:

```
openssl s_client -connect node1:8022 -starttls ftp
```

A client session with `lftp`:

```
lftp -u <access-key> -e '
  set ftp:ssl-force true;
  set ftp:ssl-protect-data true;
  open ftp://storage.example.com:8022
'
```

For the self-signed test certificate only, add `set ssl:verify-certificate no`
to the session (never in production), or install the generated certificate in
the client trust store.

Air-gapped deployment
---------------------

1. On a machine with internet access, download, verify and extract the binary:
   ```
   wget https://github.com/rustfs/rustfs/releases/download/1.0.5/rustfs-linux-x86_64-gnu-v1.0.5.zip
   wget https://github.com/rustfs/rustfs/releases/download/1.0.5/SHA256SUMS
   sha256sum -c --ignore-missing SHA256SUMS
   unzip rustfs-linux-x86_64-gnu-v1.0.5.zip   # produces ./rustfs
   ```
2. Copy the extracted `rustfs` binary to the Ansible **control node** at
   `{{ rustfs_download_local_dir }}/rustfs` (default `~/.cache/rustfs/rustfs`;
   create the directory with mode `0700` if it does not exist).
3. Run the role with `rustfs_enable_download_binary: false`. The role copies the
   pre-staged binary to every target host - no internet access required on
   the targets or control node. Checksum verification only applies to online
   downloads, so verify the archive manually as shown above.

Example Playbook
----------------

`rustfs_access_key` and `rustfs_secret_key` have **no defaults**; supply them
from a vault or the environment. The role asserts on them before doing any
work.

```yaml
- name: Deploy RustFS
  hosts: rustfs_cluster
  roles:
    - role: rustfs
      vars:
        rustfs_access_key: "{{ vault_rustfs_access_key }}"
        rustfs_secret_key: "{{ vault_rustfs_secret_key }}"
        rustfs_rpc_secret: "{{ vault_rustfs_rpc_secret }}"
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

### Upgrading to 1.0.5

- Complete the upgrade on **every** node before adding or removing pools, or
  before retrying a decommission operation.
- The security checklist mentions rolling restarts, but RustFS distributed
  clusters require all nodes to restart together (see the note above); follow
  the release-specific upgrade notes.
- While 1.0.4 and 1.0.5 nodes coexist, DELETE requests served by a 1.0.5 node
  may temporarily return HTTP 503. Retry them once every node runs 1.0.5.
- Startup readiness now waits for a strict majority of each erasure set's
  peers to re-admit the node (fails open when peers are unreachable or still
  on 1.0.4). The unit's `TimeoutStartSec` is raised to 300s to cover the
  default 120s budget; keep `rustfs_startup_readiness_max_wait_seconds` below
  that value, or disable the gate with
  `rustfs_startup_peer_readmission_enable: false`.
- 1.0.4 refreshed certificate verification and the bundled trust roots. The
  Linux zip uses the system trust store, so the role now installs
  `ca-certificates` by default.

Testing and CI
--------------

```bash
# Lint (production profile) and YAML style
ansible-lint
yamllint .

# Syntax check of the reference playbook
ansible-playbook --syntax-check -i tests/inventory tests/install_rustfs.yml

# Multi-node deployment (needs the Vagrant nodes and credentials in the env)
export RUSTFS_ACCESS_KEY=... RUSTFS_SECRET_KEY=... RUSTFS_RPC_SECRET=...
ansible-playbook -i tests/inventory tests/install_rustfs.yml
```

The same checks run in `.pre-commit-config.yaml` and in the GitHub Actions
workflow (`.github/workflows/ci.yml`).

The reference playbook lives in [`tests/install_rustfs.yml`](tests/install_rustfs.yml)
and resolves the role through the `tests/roles/rustfs` symlink, so no Galaxy
install is needed. Idempotency is verified by running the play twice and
expecting `changed=0` on the second run.

> **Molecule is intentionally not configured.** The role needs a multi-node
> cluster, XFS-formatted data disks and a running systemd, none of which a
> default Molecule container scenario provides. Testing is done against the
> Vagrant cluster above; the lint/syntax layers run in CI.

Troubleshooting
---------------

### Legacy `.minio.sys` probe errors

On a native RustFS layout (`.rustfs.sys`), a multi-node cluster may log a burst
of errors like this during startup:

```json
{"level":"ERROR","message":"internode rpc request failed",
 "volume":".minio.sys","path":"format.json","error":"volume not found"}
```

This is the upstream legacy-MinIO migration probe: RustFS checks each disk once
for `.minio.sys/format.json`, and a missing legacy volume is the expected result
for a native RustFS cluster. The internode RPC layer logs the expected miss at
`ERROR` before the caller treats it as "no legacy format". The burst is
startup-only and safe to ignore when the readiness endpoint is healthy:

```bash
curl -s http://127.0.0.1:9000/health/ready
```

Do **not** create a fake `.minio.sys/format.json` to silence the message; it can
be mistaken for migration evidence. A single
`HTTP transport failed ... Cancelled` message around a restart is also
transient (a client or peer disconnected during the restart).

License
-------

BSD

Author Information
------------------

Created for the vhkt automation repository.
