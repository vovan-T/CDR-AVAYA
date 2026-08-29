# CDR AVAYA Load and Failure Test Report

**English** | [Русский](LOAD_TEST_REPORT_2026-07-16_RU.md)

- Test date: July 16, 2026
- System under test: CDR AVAYA 1.1.13 with an experimental ACD Load Balancer and disk-backed listener spool
- Test stand: VMware virtual machine
- Result: tests passed; the prototype was accepted for integration into the main development line

## 1. Objectives

The tests evaluated:

- PostgreSQL throughput while writing RAW CDR in batches;
- listener and processor behavior up to an offered rate of 20,000 CDR records/s;
- fair processing across multiple ACD queues;
- UCID-chain processing without missing or duplicated calls;
- preservation of accepted TCP CDR while PostgreSQL is unavailable and after listener termination;
- duplicate protection when the disk journal is replayed.

## 2. CDR AVAYA versions

Testing started from installed system version `1.1.13`.

| Module | Test version |
|---|---:|
| System | 1.1.13 |
| UI | 1.1.13 |
| API | 1.1.13 |
| Updater | 1.1.12 |
| Database | 1.1.11 |
| Listener | 1.1.11 plus experimental prototype |
| Processor | 1.1.11 plus experimental prototype |

The experimental changes did not have a separate release version and were not part of the public 1.1.13 installer at test time.

## 3. Hardware

| Parameter | Value |
|---|---|
| Platform | VMware Virtual Platform, full virtualization |
| Architecture | x86_64 |
| Host CPU visible to VM | AMD Ryzen 7 7840HS with Radeon 780M Graphics |
| VM allocation | 4 vCPU |
| VM topology | 2 sockets, 2 cores per socket, 1 thread per core |
| RAM | 8,078,094,336 bytes, approximately 7.52 GiB |
| Swap | 4,294,963,200 bytes, approximately 4.00 GiB |
| VM disk | VMware Virtual S, 50 GiB |
| Filesystem | ext4 |
| Disk utilization during test | 67%, approximately 15.9 GiB free |
| PostgreSQL container limits | no dedicated CPU or RAM limits |

## 4. Software

| Component | Version |
|---|---|
| OS | Ubuntu 24.04.4 LTS, Noble Numbat |
| Kernel | Linux 6.17.0-40-generic |
| systemd | 255.4-1ubuntu8.16 |
| Python | 3.12.3 |
| psycopg2-binary | 2.9.9 |
| AsyncSSH | 2.18.0 |
| Node.js | 18.19.1 |
| npm | 9.2.0 |
| Node PostgreSQL driver `pg` | 8.22.0 |
| Vue | 3.5.39 |
| Vite | 5.4.21 |
| TypeScript | 5.9.3 |
| Docker Engine | 29.1.3 |
| Docker Compose | 2.40.3 |
| PostgreSQL | 16.14, image `postgres:16-alpine` |

Runtime dependencies were installed on the target. Frontend build dependencies were not installed in the production directory; the UI ran from a prebuilt `dist` directory.

## 5. Prototype changes

- Direct indexed CM lookup by `ucid` instead of `NULLIF(ucid, '')`.
- Batched Calls insertion and RAW status updates.
- Caching of immutable table metadata.
- One independent queue per ACD.
- Round-robin ACD Load Balancer.
- Adaptive batches of 1, 50, 500, 1,000, 5,000, and 10,000 records.
- Batched SM, EQ, and Other processing.
- Per-ACD disk journal at `$CDR_HOME/data/spool/<acd>.jsonl`.
- Atomic checkpoint after successful PostgreSQL commit.
- Replay of unprocessed journal records after listener restart.
- Stable TCP-record identifier for duplicate protection.
- Automatic journal compaction after queue release and 32 MiB growth.

## 6. Baseline measurements

Before optimization, the test dataset showed:

- listener with the previous processor running: approximately 330 RAW records/s;
- CM processor: approximately 100 RAW records/s;
- UCID lookup through `NULLIF(ucid, '')`: 8.78 ms at 45,000 RAW records;
- direct indexed `ucid = value` lookup: 0.08 ms.

## 7. PostgreSQL batch-write test

Every run inserted 10,000 rows into an isolated test table. The source CDR table was not modified.

| Batch size | Transactions | Time, s | Records/s |
|---:|---:|---:|---:|
| 1 | 10,000 | 8.827 | 1,133 |
| 5 | 2,000 | 1.892 | 5,285 |
| 30 | 334 | 0.427 | 23,415 |
| 100 | 100 | 0.251 | 39,800 |
| 500 | 20 | 0.159 | 63,003 |
| 1,000 | 10 | 0.147 | 68,108 |

Single-row INSERT limited throughput to approximately 1,100 records/s. Batches of 500-1,000 rows increased isolated PostgreSQL throughput to approximately 63,000-68,000 records/s.

## 8. Load tests

Synthetic CM records used the active customized layout. Rates below describe short controlled bursts, not a 24-hour soak test.

| Scenario | Result |
|---|---|
| One ACD, 10,000 records/s for 5 seconds | 50,000 accepted, no loss, queue drained completely |
| Two ACDs, 5,000 records/s each | 25,000 accepted per ACD; queues grew and drained evenly |
| Two ACDs, 10,000 records/s each | 50,000 accepted per ACD, 100,000 total |
| Combined 20,000 records/s | peak queue approximately 30,600 records per ACD; both drained |
| CPU at combined 20,000 records/s | peak total utilization approximately 53% |
| One ACD with spool, 10,000 records/s | 50,000 accepted; peak queue approximately 12,000, then zero |
| CPU during spool test | peak total utilization approximately 61% |

Call-chain validation:

- 5,000 transfer RAW records produced exactly 1,000 Calls;
- every resulting call contained five linked RAW IDs;
- an additional 1,000-record run produced exactly 200 Calls.

## 9. Spool failure test

The controlled PostgreSQL outage and listener failure test used this sequence:

1. PostgreSQL stopped at checkpoint `70000`.
2. The listener accepted 5,000 additional records at 5,000 records/s.
3. The journal contained exactly 5,000 new lines while the checkpoint remained `70000`.
4. The listener was terminated with `SIGKILL`.
5. PostgreSQL and the listener restarted; pending journal data was replayed.
6. The checkpoint advanced to `75000`.
7. RAW contained exactly 5,000 new records and the queue drained completely.
8. Calls contained 5,000 corresponding results.

For the idempotency check, the checkpoint was manually returned from `75000` to `70000`. The listener replayed the same 5,000 lines, but RAW counts did not change. No duplicates were created and the checkpoint returned to `75000`.

## 10. fsync mode

- Prototype operating mode: batched `fsync` every 100 ms.
- This mode sustained 10,000 records/s and recovered after process termination.
- `fsync=always` reduced throughput to approximately 2,035 records/s.
- A complete power loss could theoretically lose the last approximately 100 ms of journal data.
- Data still present only in TCP buffers cannot be fully guaranteed without source-side acknowledgment and retransmission.

## 11. Automated checks

- Python: 132 tests passed.
- Additional parameterized checks: 12 passed.
- Journal compaction preserved pending records.
- All `cdr-*` services were active after testing.
- API health: `status=ok`, `db=ok`.
- PostgreSQL container: `healthy`.
- Both test ACD queues were empty after completion.

## 12. Limitations

- Two concurrent ACDs were tested, not the planned maximum of 50.
- Peak load was applied in short bursts; no 24-hour soak test was performed.
- Long-term disk exhaustion caused by continuously growing spool was not tested.
- Physical power loss of the VM or storage was not reproduced.
- The synthetic generator validated the CDR path but not every property of a real CM network channel.
- Results apply to the documented VM and are not a universal guarantee for different CPU, storage, or PostgreSQL configurations.

## 13. Conclusion

The prototype removed the main single-row insertion and sequential lookup bottlenecks. The VMware stand demonstrated:

- one ACD at an offered 10,000 records/s;
- short combined bursts of 20,000 records/s from two ACDs;
- fair service of multiple ACD queues;
- recovery of accepted CDR after database outage and listener `SIGKILL`;
- journal replay without duplicate records.

The recommended next steps were integration into the main codebase, complete regression testing, long-duration testing, and a dedicated test with a larger number of active ACDs.
