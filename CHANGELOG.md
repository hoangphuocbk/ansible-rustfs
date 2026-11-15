# Changelog

All notable changes to this role. Version numbers follow the upstream
[RustFS releases](https://github.com/rustfs/rustfs/releases) tracked by
`rustfs_version`; role-specific changes are called out explicitly.

## Unreleased

Role hardening and simplification. No upstream version bump.

### Added

- `meta/main.yml`, `meta/argument_specs.yml` (80 validated options) and
  `requirements.yml` (ansible.posix, community.crypto).
- `rustfs_artifact_name` so the release artifact can be swapped for arm64 or
  musl builds without rewriting the download URL.
- `rustfs_iptables_input_chain`, `rustfs_iptables_output_chain` and
  `rustfs_iptables_comment` in place of the undeclared `iptable_*` variables.
- `.ansible-lint` (production profile), `.yamllint`, `.pre-commit-config.yaml`,
  `.github/workflows/ci.yml` and README sections for idempotency, check mode
  and rollback.

### Changed

- **Breaking:** `rustfs_access_key` / `rustfs_secret_key` have no defaults and
  must be supplied (from a vault, the environment, or `-e`). The role fails
  fast instead of deploying the public `rustfsadmin` pair.
- **Breaking:** `enable_download_binary` is now
  `rustfs_enable_download_binary`.
- `rustfs_download_local_dir` now defaults to `~/.cache/rustfs` (created with
  mode `0700`) instead of `/tmp`.
- `rustfs_env_extra` is now a list (it was documented as free-form text but
  iterated as a list by the template).
- Checksum verification delegates to `get_url` (a `sha256:<url>` checksum)
  instead of parsing `SHA256SUMS` in Jinja.
- Self-signed FTPS material is generated and inspected with
  `community.crypto` instead of `openssl` command tasks. The certificate
  directory now also holds `rustfs_csr.pem` when the pair is self-signed.
- Internal variables use the `__rustfs_` prefix; the resolved bind address is
  an internal fact instead of overriding the `rustfs_bind_address` default.
- Task files use the `.yml` extension; the test inventory is a structured
  directory with `group_vars/`.

### Fixed

- Check mode no longer fails when FTPS is enabled with a self-signed
  certificate.
- `template` tasks keep `backup: true`; credentials are quoted in the rendered
  environment file.
- Tasks report `changed` correctly for the self-signed certificate generation.

## 1.0.5 - 2026-11-12

Upstream: [RustFS 1.0.5](https://github.com/rustfs/rustfs/releases/tag/1.0.5),
which includes the [1.0.4](https://github.com/rustfs/rustfs/releases/tag/1.0.4)
patch.

### Added

- `rustfs_checksums_url` and `rustfs_verify_checksum`: the control node now
  downloads the release `SHA256SUMS` and passes the archive hash to `get_url`,
  so a truncated or tampered zip is rejected before it is unpacked.
- `rustfs_region` (`RUSTFS_REGION`). RustFS 1.0.5 validates the CreateBucket
  `LocationConstraint` against the configured region; the role keeps the
  upstream default (`us-east-1`) unless the variable is set.
- `rustfs_veeam_sosapi_enable` (`RUSTFS_VEEAM_SOSAPI_ENABLE`, new in 1.0.5).
  Veeam Backup & Replication SOSAPI virtual objects are enabled by default
  upstream; the role renders the switch explicitly so it can be turned off.
- `rustfs_console_oidc_session_duration`
  (`RUSTFS_CONSOLE_OIDC_SESSION_DURATION`, new in 1.0.5): configurable OIDC
  console credential lifetime and logout-token validity (1-604800 seconds;
  upstream clamps larger values).
- `rustfs_startup_peer_readmission_enable`
  (`RUSTFS_STARTUP_PEER_READMISSION_ENABLE`, new in 1.0.5) and
  `rustfs_startup_readiness_max_wait_seconds`
  (`RUSTFS_STARTUP_READINESS_MAX_WAIT_SECS`). Startup readiness now waits for
  a strict majority of each erasure set's peers to re-admit the node, and
  always fails open.
- `rustfs_internode_http_protocol` (`RUSTFS_INTERNODE_HTTP_PROTOCOL`, new in
  1.0.5), defaulting to `auto`. On the plaintext internode transport this role
  configures it keeps HTTP/1.1; if internode TLS is added later, TLS ALPN can
  negotiate HTTP/2. Set `http1` to force HTTP/1.1 everywhere.
- Scanner tuning from 1.0.5: `rustfs_scanner_walkers_per_disk`,
  `rustfs_scanner_disk_io_budget_iops`,
  `rustfs_scanner_disk_io_budget_bytes_per_sec` and
  `rustfs_scanner_recovery_backoff_cap_seconds`. All default to `null`, so the
  upstream media-aware defaults apply and the environment file stays minimal.
- Optional plain FTP and explicit FTPS (FTPES) listeners via
  `rustfs_ftp_*` and `rustfs_ftps_*`, both disabled by default.
- FTPS certificate management: the role creates and secures the certificate
  directory, validates that the certificate and key parse, are not expired and
  match, and can generate a self-signed pair with
  `rustfs_ftps_self_signed: true`.
- The test deployment now enables FTPS with a generated self-signed
  certificate.

### Changed

- `rustfs_version` and the test deployment now install **1.0.5** (gnu x86_64
  zip) instead of 1.0.3. The 1.0.4 changes are included.
- `rustfs_system_packages` now also installs `bash` and `ca-certificates`,
  matching the documented software checklist. `ca-certificates` matters for
  the 1.0.4 trust-root and certificate-verification refresh: the Linux zip
  verifies TLS peers against the system trust store.
- Kernel tuning now applies exactly the four sysctls from the upstream network
  checklist (`net.core.rmem_max`, `net.core.wmem_max`,
  `net.ipv4.tcp_keepalive_time`, `net.ipv4.tcp_slow_start_after_idle`). The
  undocumented `vm.swappiness` and `net.core.somaxconn` settings were removed;
  re-add them through `rustfs_sysctl_settings` if your environment needs them.
- The README now maps the role to the upstream pre-installation checklists
  (software, network, hardware and security): it documents what the role
  handles and the operator-owned steps (time sync/DNS, disabling `mlocate` /
  `auditd` / antivirus, SELinux/AppArmor, firewall, TLS certificates,
  monitoring, backups and rollback). The OS/kernel requirement now says Linux
  5.10+ is recommended (upstream supports Ubuntu 20.04+ and RHEL 8/9), and the
  disk guide recommends mounting by filesystem UUID as upstream does.
- The systemd unit raises `TimeoutStartSec` from 120s to 300s so it covers
  RustFS initialization plus the default 120s peer re-admission budget.
- Control-node downloads now verify TLS certificates by default through
  `rustfs_download_validate_certs` (previously hard-coded
  `validate_certs: false`). Set it to `false` only for a private mirror with a
  self-signed certificate.
- Added `rustfs_bind_address`, derived from `rustfs_bind_interface` by default,
  and reused it for the test deployment's listener addresses and the optional
  firewall rules instead of repeating the interface-fact expression.
- Aligned the systemd unit with upstream: `rustfs server` startup, journal
  stdout/stderr, `/opt/rustfs` working directory, `ProtectSystem=full`,
  `ReadWritePaths` covering the configured data and log directories, and a
  45-second stop timeout.
- Documented the benign startup-only `.minio.sys` legacy-format probe errors
  and the transient `HTTP transport failed ... Cancelled` messages.

### Security

- Archive checksum verification is enabled by default.
- Download TLS verification is enabled by default.
- Plain FTP is disabled by default and binds to `127.0.0.1:8021` when enabled;
  upstream defaults to `0.0.0.0:8021`.
- Generated self-signed certificates use 4096-bit RSA, SHA-256, a 365-day
  validity period, explicit SANs, `CA:false`, `serverAuth` EKU and key
  identifiers. The key is `0600` inside a `0700` directory, has no passphrase
  (the service must read it unattended), and is renewed 30 days before expiry.
- Self-signed certificates remain opt-in and are documented as test-only; the
  role never disables certificate verification on clients.
- 1.0.5 restricts root-derived service-account creation/import and removes
  credential fields from rejected-key diagnostics. No role change is required,
  but this reinforces keeping the root credential pair non-default.
- Upstream flipped `RUSTFS_TRUSTED_PROXY_ENABLE_RFC7239` to `false` (legacy
  `X-Forwarded-For` is used unless RFC 7239 is explicitly selected). The role
  does not set it; set it via `rustfs_env_extra` only behind a trusted proxy.

### Fixed

- The test play now defines the Vagrant host-only `eth1` interface only once
  and asserts that it exists, fixing the undefined `ansible_enp0s8` error.

### Upstream 1.0.4 (included)

- Refreshed certificate verification and bundled trust roots, including the
  removal of the distrusted emSign ECC Root CA - G3 from the container images,
  plus a dependency refresh. No server configuration changes and no role
  variable changes.

### Upstream 1.0.5 notable changes

- **Upgrade ordering** - finish the upgrade on every node before adding or
  removing pools, or retrying a decommission.
- **Mixed-version DELETEs** - 1.0.5 nodes can return HTTP 503 for DELETE while
  1.0.4 nodes are still in the cluster; retry after the full upgrade.
- **Veeam SOSAPI** - authenticated GET/HEAD of the two SOSAPI keys returns
  generated XML by default (`RUSTFS_VEEAM_SOSAPI_ENABLE`).
- **Console OIDC session duration** - configurable credential and logout-token
  lifetime (`RUSTFS_CONSOLE_OIDC_SESSION_DURATION`).
- **IAM and policy** - wildcard support in IAM policy resources,
  `s3:if-match` / `s3:if-none-match` condition keys, and object-creation
  operation conditions.
- **S3 API** - conditional bulk deletes, bucket location validation (now
  enforced against `RUSTFS_REGION`), CORS for virtual-hosted-style requests,
  and stricter multipart/listing error semantics.
- **Startup readiness** - peer re-admission gate before a node reports ready.
- **Scanner** - bucket walkers decoupled from disk count, per-disk I/O budgets
  (media-aware `auto` default), a configurable recovery backoff cap, and
  checkpoint/usage continuity fixes.
- **Heal / decommission** - MRF lifecycle V2 checkpoints and reader capability
  are opt-in via `RUSTFS_MRF_LIFECYCLE_V2_ENABLE`; only enable it after every
  reader and the rollback binary are on 1.0.5. Decommission and heal retries
  were substantially reworked.
- **Internode transport** - `RUSTFS_INTERNODE_HTTP_PROTOCOL=auto` enables
  TLS ALPN negotiation (HTTP/2 over TLS, HTTP/1.1 on plaintext). The HTTP/2
  tuning knobs stay inert without TLS. The role now defaults to `auto`.
- **Observability** - cgroup limit sampling, request entry-to-dispatch and
  response-body completion metrics, scanner/heal walk metrics, and allocator
  reclaim totals.
- **Audit / notify** - admission rate limits for authorization-denial audit
  entries (`RUSTFS_AUDIT_DENIAL_RATE`, `..._ANONYMOUS_RATE`) and a 1 GiB
  per-target queue byte budget (`RUSTFS_TARGET_STORE_MAX_BYTES`). The role does
  not configure audit or notify targets; pass these through `rustfs_env_extra`
  when you do.
- **Idle drive monitoring** - the scanner's idle-drive probes now honor the
  upstream opt-out.
- **Performance** - batched listings and metacache streams, bounded xl.meta
  read workers with HEAD early-stop, fewer metadata clones, and lower
  delete-recovery head-of-line blocking.

### Upgrade notes

- Run the play against the whole cluster and restart every node together;
  RustFS forbids rolling restarts.
- Complete the 1.0.5 upgrade on all nodes before any pool add/remove or
  decommission retry, and retry DELETE calls that returned 503 while versions
  were mixed.

## 1.0.3 - 2026-10-17

Upstream: [RustFS 1.0.3](https://github.com/rustfs/rustfs/releases/tag/1.0.3)

### Role

- The `rustfs_version` default and the test deployment now install 1.0.3
  (gnu x86_64 zip) instead of 1.0.1.
- Release notes moved from `README.md` to this file.

### Upstream

- No server code or server configuration changes from 1.0.2.
- The embedded Console is updated to
  [v0.1.37](https://github.com/rustfs/console/releases/tag/v0.1.37): it hides
  the duplicate bucket heading at the bucket root, adds Vietnamese
  localization, and improves deleted-object highlighting and filtering.

### Upgrade notes

- Drop-in upgrade from 1.0.2. RustFS forbids rolling restarts, so run the play
  against the whole cluster and let every node restart together.

## 1.0.2 - 2026-10-16

Upstream: [RustFS 1.0.2](https://github.com/rustfs/rustfs/releases/tag/1.0.2)
(179 commits over 1.0.1)

### Configuration changes

- **HTTP/3 (opt-in)** - `RUSTFS_HTTP3_ENABLE` (default `false`) enables the new
  QUIC transport on UDP at the S3 listener address and port. It requires a
  binary built with `--features http3`, TLS via `RUSTFS_TLS_PATH` and, in the
  1.0.2/1.0.3 code, the embedded console disabled. The console and internode
  RPC remain on TCP. Upstream release zips (including the one this role
  installs) are **not** built with the `http3` feature.
- **TLS certificate publication** - `RUSTFS_TLS_PUBLISH_DIR` names a dedicated
  persistent directory (mode `0700`, owned by the RustFS user) for externally
  driven certificate publication. It is used together with
  `RUSTFS_TLS_RELOAD_ENABLE` (default `false`) and `RUSTFS_TLS_RELOAD_INTERVAL`
  (default `30` seconds, minimum `5`). This role does not manage TLS material,
  so these settings apply only to hand-managed deployments.
- **S3 Tables maintenance scheduler** -
  `RUSTFS_TABLE_MAINTENANCE_SCHEDULER_ENABLED` (default `false`),
  `..._BUCKETS` (required when enabled; 1-32 distinct buckets) and
  `..._INTERVAL_SECONDS` (default `300`; 30-86400). Standalone server only; it
  queues maintenance jobs but does not run workers.
- **Cleanup isolation (opt-in)** - `RUSTFS_CLEANUP_ISOLATE_ENABLE` (default
  `false`) plus the `RUSTFS_CLEANUP_*` and `RUSTFS_PUT_RENAME_TAIL_CLEANUP_*`
  admission/worker budgets. Configuration is read once at first use, so changes
  require a restart. On Linux the default cleanup CPU mask expects logical CPUs
  1-2 to be available.
- **GET-stage local summaries** - `RUSTFS_OBS_GET_STAGE_LOCAL_SUMMARY_ENABLED`
  (default `false`), `..._SAMPLE_RATE` (default `64`) and `..._INTERVAL_SECS`
  (default `15`). Only active when metrics export is disabled.
- **Secret keys** - the 40-byte maximum is removed. Secret keys must still be at
  least 8 bytes, and the full value is used for signing instead of being
  truncated to 40 bytes.
- **Observability** - `RUSTFS_OBS_USE_STDOUT` is now honored with per-signal
  OTLP endpoints, and telemetry `service.version` defaults to the build version
  (`RUSTFS_OBS_SERVICE_VERSION` still overrides it).
- **GNU artifacts** - Linux gnu builds now pin a glibc 2.31 ABI floor
  (Ubuntu 20.04+/RHEL-Rocky 9 compatible). Ubuntu 24.04 is unaffected.

### Other notable changes

- S3 Tables: durable-strong diagnostics, catalog backups, disabling empty
  catalogs and durable-strong table maintenance.
- Quota: sharded durable reservation protocol.
- Event notifications: Kafka SASL without TLS.
- WebDAV reports quota properties, S3 Select supports quoted record delimiters,
  and object lock exposes remaining retention days.
- Performance: lower multipart/object I/O overhead, lower scanner/checkpoint
  allocations, indexed heal history reconciliation and request watchdog fixes.
- Reliability fixes across heal, scanner, replication, delete transaction
  recovery, OIDC, TLS reload and multipart staging.

### Role

- No new role variables: the new features above are either build-time or
  outside this role's standalone, console-enabled deployment. Advanced opt-ins
  can be passed through `rustfs_env_extra`.
- This role update goes directly from 1.0.1 to 1.0.3, so the 1.0.2 changes above
  are included in the 1.0.3 bump.

### Upgrade notes

- Restart all nodes together. Secret keys longer than 40 bytes that were
  previously rejected now work; keys shorter than 8 bytes still fail.

## 1.0.1 - 2026-10-03

Upstream: [RustFS 1.0.1](https://github.com/rustfs/rustfs/releases/tag/1.0.1)

The role tracks the upstream 1.0.1 release (385 commits over 1.0.0). Notable
changes relevant to this deployment:

- **io_uring read path** - integrated backend improvements plus a new
  per-operation size cap `RUSTFS_IO_URING_READ_CHUNK_BYTES` (4 KiB-128 MiB),
  exposed here as `rustfs_io_uring_read_chunk_bytes`.
- **Integrity tooling** - inventory, audit and protected migration.
- **S3 Tables** - support for staged table creation.
- **Event notifications** - exposes the subscriptions used by event
  destinations.
- **Performance** - signed PUTs skip a redundant content-sha256 pass, multipart
  I/O setup and metadata reads are parallelized, and S3 Select `ScanRange`
  scanning is faster.
- **Packaged artifacts** - Linux `.deb` and `.rpm` packages are published for
  both `gnu` and `musl` on amd64/arm64, and the CLI is now included in the
  Linux zip artifacts this role uses.
- **Reliability fixes** across heal, scanner, replication, readiness and
  decommission paths - see the release notes for the full list.
- **Enterprise Connect diagnostics** (offline CPU/network/lock/API profiling and
  signed health jobs) - not used by this standalone role.

Beyond the 1.0.1 changes, the role also models the console URL prefix
(`rustfs_console_prefix`) and per-service CORS allow-lists
(`rustfs_cors_allowed_origins`, `rustfs_console_cors_allowed_origins`). Both
services are **same-origin only by default**, so set an allow-list for
cross-origin browser access.
