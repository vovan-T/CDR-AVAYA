# CDR AVAYA

**English** | [Русский](README_RU.md)

CDR AVAYA is a web platform for receiving, processing, storing, and analyzing call detail records from Avaya systems.

Base installation: **2.5.0**. Current cumulative update: **2.5.4**.

## Product tour

### Call journal

![CDR AVAYA call journal](https://vovan-t.github.io/CDR-AVAYA/images/cdr-avaya-dashboard.png)

### Call route and linked source records

![CDR AVAYA linked call details](https://vovan-t.github.io/CDR-AVAYA/images/cdr-avaya-details.png)

### System diagnostics

![CDR AVAYA system diagnostics](https://vovan-t.github.io/CDR-AVAYA/images/cdr-avaya-diagnostics.png)

## Highlights

- receives Avaya Communication Manager CDR over TCP;
- supports configurable `customized` and unformatted CM layouts;
- preserves original RAW records separately from the normalized call journal;
- reconstructs CM call chains by UCID;
- imports Session Manager and Equinox Management XML records;
- accepts Avaya SBCE CDR files through the shared SFTP upload account;
- supports multiple independent CM, SM, EQ, SBCE, and Other sources per ACD;
- provides dictionaries for extensions, trunks, routes, VDNs, and addresses;
- offers filters, full-text search, CSV export, RAW mode, and linked record details;
- keeps separate `View` column catalogs for CM, SM, EQ, SBCE, and Other;
- displays linked SM/SBCE RAW details and SBCE routes from `Routing Profile`;
- includes service, source, PostgreSQL, Docker, storage, and network diagnostics;
- provides users, per-ACD permissions, audit history, Support Pack, Backup/Restore, and update rollback;
- supports online and fully offline Docker deployment on Ubuntu and Astra Linux.

## Quick installation

On a Linux computer with Internet access:

```bash
wget https://github.com/vovan-T/CDR-AVAYA/releases/latest/download/install.sh
chmod +x install.sh
sudo ./install.sh
```

Choose:

- `online` to install the release directly on the current server;
- `offline` to download the three files required by an isolated server.

The installer deploys `cdr-app` and `cdr-db`. A demonstration ACD named `CM1`, listening on TCP port `5001`, is available after the first start.

See the [Installation Guide](INSTALL.md) and the complete manuals:

- [English PDF manual](docs/cdr-avaya-2.5-manual-en.pdf)
- [Русское PDF-руководство](docs/cdr-avaya-2.5-manual-ru.pdf)
- [Release Notes](docs/RELEASE_NOTES.md)

## Supported sources

### Communication Manager

Receives CDR over TCP, stores every original record, and reconstructs related call stages into one journal entry by UCID. Customized fields, unformatted records, trunks, VDNs, directions, and dictionaries are supported.

### Session Manager

Downloads extended XML CDR from one or more Session Manager servers. The system preserves RAW XML-derived data, builds a dedicated journal, and identifies the source server.

### Equinox Management

Imports conference XML records and participants. A root-side staging script copies completed and changing CDR files into the `CDR_User` home area for synchronized collection.

### Session Border Controller for Enterprise

Receives SBCE CDR files in isolated instance folders. Multiple SBCE instances can coexist inside one ACD. Linked RAW details, `Routing Profile`, and `Server Flow` are available in the UI.

### Other

A configurable source for third-party systems. It can receive TCP records, read local files, or fetch data over SCP/SFTP using Flat or XML formats with user-defined fields.

## Verified release path

The public 2.5.4 release was verified on Ubuntu using the exact sequence:

```text
clean Docker 2.5.0 -> cumulative update 2.5.4 -> rollback -> 2.5.4 -> commit
```

The same-version PostgreSQL Backup/Restore path, `CDR_User` password restoration, SFTP login, browser navigation, Docker diagnostics, API health, and published SHA-256 files were also verified.

## Load testing

The VMware test stand processed 50,000 records at an offered rate of 10,000 records/s without loss and recovered queued records after PostgreSQL and listener failures. These are short synthetic test results for the documented hardware, not a universal capacity guarantee.

[Read the full load-test report](docs/LOAD_TEST_REPORT_2026-07-16.md).

## Project status

CDR AVAYA 2.5 is distributed as ready-to-run Docker images and an offline bundle. The application source is maintained privately; this public repository contains documentation, release assets, screenshots, and test reports.

## Contact

Author: [vovan-T](https://github.com/vovan-T)

Use [GitHub Issues](https://github.com/vovan-T/CDR-AVAYA/issues) for questions, deployment feedback, and feature requests.
