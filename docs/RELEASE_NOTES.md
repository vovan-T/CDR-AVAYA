# CDR AVAYA 2.5 Release Notes

**English** | [Русский](RELEASE_NOTES_RU.md)

Base installation: **2.5.0**
Current cumulative update: **2.5.4**
Database compatibility version: **1.0**

## 2.5.4

- Added one consistent telecom background to sign-in, call journal, and system administration views.
- Preserved contrast for tables, panels, filters, terminals, and operational controls.
- Updated public screenshots and English/Russian documentation.
- Kept the update cumulative: it installs directly over a clean Docker 2.5.0 deployment.
- Verified the complete Ubuntu path: `2.5.0 -> 2.5.4 -> rollback -> 2.5.4 -> commit`.
- Verified same-version Backup/Restore, `CDR_User` password restoration, SFTP authentication, browser navigation, Docker diagnostics, and release checksums.

## Included cumulative changes

### Source-specific View catalogs

- CM, SM, EQ, SBCE, and Other expose their own available field catalogs.
- SBCE includes `Routing Profile` and `Server Flow`.
- SM includes its extended XML call fields.
- EQ keeps conference and participant-specific fields.

### SM and SBCE Details

- SM and SBCE use the shared linked-RAW Details window.
- SBCE grouping prefers Session ID, then UCID, Call ID, and a conservative fallback.
- Restoring older settings repairs mandatory SM/SBCE Details actions.

### SBCE route presentation

- The Calls route is derived from `Routing Profile`.
- `Routing Profile` and `Server Flow` are selected from the same best completed RAW row as the route label.
- Historical Calls are not rewritten automatically; administrators can rebuild them from preserved RAW with `recalculate.sh`.

### Docker runtime

- Individual service start, stop, and restart actions target the corresponding supervised child process.
- External EQ/SM/SBCE fetch failures no longer restart the whole application container.
- Update restarts are queued through the Docker supervisor.
- Post-update diagnostics distinguish ACD table prefixes correctly.

### Update safety

- Update archives require the matching SHA-256 sidecar in CLI, API, and UI flows.
- Update 2.5.4 creates rollback mode `3`: application files plus a separate database copy.
- `Rollback` restores the previous manifest, filesystem, database, and `CDR_User` account state.
- `Commit` removes rollback files and the temporary rollback database.

### Backup, Restore, and Migration Tool

- Standard Backup creates a validated full PostgreSQL custom-format dump and Backup manifest.
- Standard Restore replaces the entire database only when Backup DB version exactly matches System DB version.
- Cross-version and selective recovery belong to Migration Tool, not standard Restore.
- Docker Backup/Restore preserves the host `CDR_User` password hash through a restricted host bridge.

### Administration

- Added emergency administrator password reset for Docker deployments.
- Added expanded service diagnostics, Support Pack, users, per-ACD permissions, and audit history.
- Documentation now covers all CM, SM, EQ, SBCE, Other, and system settings fields.

## 2.5.0 baseline

CDR AVAYA 2.5.0 introduced the supported Docker deployment model:

- `cdr-app` and PostgreSQL 16 `cdr-db` containers;
- fresh database baseline without upgrade migrations during installation;
- demonstration ACD `CM1`, default listener port `5001`;
- shared SFTP upload account `CDR_User` and initial folder `sbce/1`;
- online and offline installation from one public release;
- Ubuntu and Astra Linux deployment support;
- System manifest, Backup manifest, Update manifest, and Rollback manifest;
- DB compatibility baseline `1.0`.

## Compatibility

| Component | Value |
|---|---|
| Base installation | Docker CDR 2.5.0 |
| Current update | 2.5.4 cumulative |
| Supported system line | 2.5.x |
| Current DB version | 1.0 |
| Standard Restore | exact DB version only |
| Architectures | x86-64 |
| Tested operating systems | Ubuntu Server 24.04, Astra Linux 1.7 |

## Checksums

```text
009147f83f0b500f70730f5288e155e9a51dadb3721281da151cb3db39172f9b  cdr-avaya-docker-images-2.5.0.tar.gz
82f0d5900f2d9a64cd1733c0dd195a7c303b0221e2046fbac9d22ce59f2d2d3b  update-2.5.4.tar.gz
```

See the complete [English manual](cdr-avaya-2.5-manual-en.pdf) or [Russian manual](cdr-avaya-2.5-manual-ru.pdf).
