# CDR AVAYA 2.5 Installation Guide

**English** | [Русский](INSTALL_RU.md)

CDR AVAYA 2.5 is deployed as two Docker containers:

- `cdr-app` contains the API, web interface, listeners, processors, file collector, and maintenance services;
- `cdr-db` contains PostgreSQL 16 and the CDR database.

The installer deploys the application only. ACD and telephony source configuration is completed later in the web interface.

## Requirements

- x86-64 Linux host;
- Ubuntu Server 24.04 or Astra Linux 1.7;
- Docker Engine and Docker Compose v2;
- root or sudo access;
- at least 4 CPU cores, 8 GB RAM, and 40 GB free disk space for a practical production starting point;
- open TCP port `8010` for the web interface;
- open source-specific ports such as CM listener port `5001` and SSH/SFTP port `22` when required.

Verify Docker before installation:

```bash
docker --version
docker compose version
sudo docker ps
```

If Docker or Compose is missing, stop and install the distribution-supported packages before running the CDR installer.

## One-file bootstrap

On a Linux computer with Internet access:

```bash
wget https://github.com/vovan-T/CDR-AVAYA/releases/latest/download/install.sh
chmod +x install.sh
sudo ./install.sh
```

The script asks for one mode:

- `online` downloads the release and installs it on the current server;
- `offline` downloads the three files that must be transferred to an isolated server.

## Online installation

Select `online` in `install.sh`. The script downloads the published release files, validates their checksum, and starts the regular installer.

The installation asks for:

- the Web/API port, default `8010`;
- the `CDR_User` password, entered twice.

The installation path is fixed at `/opt/cdr` for the 2.5 release line.

## Offline installation

Select `offline` on a computer with Internet access. Transfer these files to the isolated server:

```text
install-offline.sh
cdr-avaya-docker-images-2.5.0.tar.gz
cdr-avaya-docker-images-2.5.0.tar.gz.sha256
```

On the target server:

```bash
sha256sum -c cdr-avaya-docker-images-2.5.0.tar.gz.sha256
chmod +x install-offline.sh
sudo ./install-offline.sh
```

The offline installer loads both Docker images locally. It does not contact GHCR or Docker Hub.

## What the installer creates

```text
/opt/cdr/
├── backup/
├── data/
│   └── sbce/1/
├── logs/
├── update/
├── .ssh/
├── docker-compose.yml
├── docker-compose.db.yml
├── manifest.env
└── .env
```

It also creates:

- host account `CDR_User` for systems that upload CDR files over SFTP;
- host group `cdr`;
- restricted SFTP configuration;
- host bridge units used to preserve and restore `CDR_User` account state during Backup/Restore;
- Docker volume `cdr_cdr-db-data` for PostgreSQL.

The SFTP root is `/opt/cdr/data`. The first SBCE instance writes to path `sbce/1`, while the SBCE UI field `Path` contains only `1`.

## First start

Open:

```text
http://SERVER_IP:8010
```

Initial application credentials:

```text
Login:    admin
Password: admin
```

The interface requires an administrator password change at first login.

A demonstration ACD named `CM1` is created automatically. Its default CM listener port is `5001`. The demonstration records, dictionary, fields, and display rules can be inspected or the ACD can be deleted after creating a production ACD.

## Verification

```bash
cd /opt/cdr
sudo docker compose ps
curl -sS http://127.0.0.1:8010/api/health
sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh post-update-check 2.5.0
```

Expected result:

```text
cdr-app   healthy
cdr-db    healthy
{"status":"ok","db":"ok"}
STATUS: OK
```

Check listening ports:

```bash
sudo ss -lntp | grep -E ':8010|:5001|:22'
```

## Installing cumulative update 2.5.4

Download both files from the release:

```text
update-2.5.4.tar.gz
update-2.5.4.tar.gz.sha256
```

Copy them into `/opt/cdr/update`, verify the checksum, and use `System -> Updates`:

```bash
sudo cp update-2.5.4.tar.gz update-2.5.4.tar.gz.sha256 /opt/cdr/update/
cd /opt/cdr/update
sudo sha256sum -c update-2.5.4.tar.gz.sha256
```

The update is cumulative and can be installed directly over clean Docker 2.5.0. It creates a rollback snapshot according to the update manifest. Use `Rollback` to return to the previous state or `Commit` to accept the update and remove rollback data.

CLI equivalents:

```bash
sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh \
  update /opt/cdr/update/update-2.5.4.tar.gz

sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh rollback-status
sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh rollback
sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh update-commit
```

## HTTPS

CDR intentionally serves HTTP. Place nginx or another reverse proxy in front of port `8010` for production HTTPS:

```text
Browser -> HTTPS reverse proxy -> http://127.0.0.1:8010
```

Do not expose port `8010` directly to an untrusted network.

## Backup and Restore

Create a full same-version PostgreSQL backup:

```bash
sudo docker exec cdr-app /opt/cdr/bin/cdr-backup.sh --database
```

Restore it:

```bash
sudo docker exec cdr-app /opt/cdr/bin/cdr-backup.sh \
  --restore /opt/cdr/backup/cdr-db-HOST-YYYYMMDD-HHMMSS.tar.gz \
  --confirm RESTORE --yes
```

Standard Restore requires exact equality between Backup DB version and System DB version. Use the documented Migration Tool process for a different DB version.

## Password recovery

If an administrator cannot sign in:

```bash
sudo docker exec -it cdr-app \
  /opt/cdr/venv/bin/python /opt/cdr/bin/cdr-reset-password.py admin
```

## Logs and Support Pack

Application logs are stored under `/opt/cdr/logs`. In the UI, open `System -> Diagnostics -> Support Pack` to create an archive for technical support.

Useful commands:

```bash
sudo docker logs --tail 120 cdr-app
sudo docker logs --tail 120 cdr-db
sudo docker exec cdr-app /opt/cdr/bin/cdr-control.sh docker-diagnostics
```

## Next steps

Continue with the [English PDF manual](docs/cdr-avaya-2.5-manual-en.pdf) for Communication Manager, Session Manager, Equinox Management, SBCE, Other, administration, Backup/Restore, and troubleshooting.
