# Changelog

All notable changes to this role. Version numbers follow the upstream
[RustFS releases](https://github.com/rustfs/rustfs/releases) tracked by
`rustfs_version`; role-specific changes are called out explicitly.

## Unreleased

### Added

- Optional plain FTP and explicit FTPS (FTPES) listeners via
  `rustfs_ftp_*` and `rustfs_ftps_*`, both disabled by default.
- FTPS certificate management: the role creates and secures the certificate
  directory, validates that the certificate and key parse, are not expired and
  match, and can generate a self-signed pair with
  `rustfs_ftps_self_signed: true`.
- The test deployment now enables FTPS with a generated self-signed
  certificate.

### Changed

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

- Plain FTP is disabled by default and binds to `127.0.0.1:8021` when enabled;
  upstream defaults to `0.0.0.0:8021`.
- Generated self-signed certificates use 4096-bit RSA, SHA-256, a 365-day
  validity period, explicit SANs, `CA:false`, `serverAuth` EKU and key
  identifiers. The key is `0600` inside a `0700` directory, has no passphrase
  (the service must read it unattended), and is renewed 30 days before expiry.
- Self-signed certificates remain opt-in and are documented as test-only; the
  role never disables certificate verification on clients.

### Fixed

- The test play now defines the Vagrant host-only `eth1` interface only once
  and asserts that it exists, fixing the undefined `ansible_enp0s8` error.

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
