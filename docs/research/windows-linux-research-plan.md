# WeChat Connector: Windows mechanism audit → Linux port

Date: 2026-10-31
Status: Research plan / preliminary source-backed findings (not yet end-to-end tested)

## Goal
First understand the actual Windows `wechatauto-replica` database-read mechanism at code level, document all reusable algorithms and version-specific assumptions, then evaluate adapting `chatlog-keeper` into a Linux-only, local, read-only WeChat connector. The first MVP is incoming-message history and incremental events; no message sending or automated login protocol.

## Sources and exact entry points
1. Windows upstream: https://github.com/fanyuantaier/wechatauto-replica/blob/main/README.zh-CN.md
2. Windows database engine: https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py
3. Linux foundation: https://github.com/labazhou2024/chatlog-keeper/blob/main/README.zh.md
4. Linux platform notes: https://github.com/labazhou2024/chatlog-keeper/blob/main/docs/linux.zh.md
5. Linux memory API: https://github.com/labazhou2024/chatlog-keeper/blob/main/chatlog_keeper/core/_linux_process_memory.py
6. Linux key flow: https://github.com/labazhou2024/chatlog-keeper/blob/main/chatlog_keeper/linux_key.py
7. Linux scanner: https://github.com/labazhou2024/chatlog-keeper/blob/main/chatlog_keeper/scripts/linux_memory_scan.c
8. Linux snapshot: https://github.com/labazhou2024/chatlog-keeper/blob/main/chatlog_keeper/core/_snapshot.py

Record immutable commit SHA for each repository before implementation and cite file/line/function for all technical claims.

## Confirmed preliminary observations (code and documentation examined)

### Windows implementation
- `wechatauto/db.py` describes locating account data through config, scanning `Weixin.exe` via `OpenProcess`, `VirtualQueryEx`, `ReadProcessMemory`, verifying candidate SQLCipher keys with page-one HMAC, and decrypting to a temporary SQLite-readable cache.
- A separate Windows *master-key* path `extract_master_key_from_cfg` finds `weixin.dll`, scans for a `global_config` landmark and follows a version-sensitive pointer chain, reads configuration key material and combines it with material extracted from a DLL instruction pattern. `derive_keys_from_master` then applies PBKDF2-HMAC-SHA512 with each database's salt and checks HMAC. The exact anchors/offsets are Windows- and version-specific and **must not be reused on Linux**.
- `db.py` uses 4096-byte pages, a page reserve of 80 bytes (IV plus HMAC), and page-one HMAC verification. These are characteristics of the targeted build / database format, not permanent cross-version guarantees.
- README describes multi-shard pitfalls: `(conversation, local_id)` is not globally unique; sender numerical identifiers must resolve within each shard's own `Name2Id` namespace; consumer messages should carry shard identity.
- The Windows project also implements UI automation and even memory modification for some unrelated UI features; these components are explicitly out of scope for the read-only connector.
- Latest examined README claims version 1.2.9 and 4.2.0.11 compatibility. This is a project claim, not independent verification by this research.

### Linux foundation
- `chatlog_keeper/core/_linux_process_memory.py` implements `process_vm_readv` read-only access and `/proc/<pid>/maps` parsing, with explicit permission failures.
- `chatlog_keeper/scripts/linux_memory_scan.c` attempts `process_vm_readv` and has a `ptrace(PTRACE_ATTACH)` / `PTRACE_PEEKDATA` fallback. Thus its scanner must **not** be described as guaranteed no-debugger/no-process-interruption in every fallback path.
- Linux documentation says Ubuntu's Yama `ptrace_scope=1` often denies passive reads of an existing unrelated process. Parent-child process topology can allow access, subject to credentials, dumpability, sandboxing and other controls.
- Linux WeChat 4.1.13.9 x86_64 is documented as validated through a startup GDB hardware breakpoint at the internal KDF boundary, not through successful passive key scanning. LD_PRELOAD observation is an additional active path if applicable.
- `_snapshot.py` implements bounded attempts to copy DB, `-wal`, `-shm` as a consistent family; it checks before/after file signatures and handles platform-specific fallbacks. Reuse should preserve fail-closed semantics.

## Stage A1: Full Windows audit (deliver `docs/windows-mechanism.md`)
1. Pin upstream commit SHA and WeChat tested versions.
2. Trace account/process matching (avoid wrong-account database/key).
3. Trace memory read permissions: `OpenProcess`, `VirtualQueryEx`, `ReadProcessMemory`; check every call site for write or debugger operations.
4. Trace two key discovery paths separately: candidate SQLCipher key scan vs. master-key recovery via DLL/config object. Document exact inputs, outputs, assumptions, validation, failure modes and version dependencies.
5. Trace cached key lifecycle: storage protection (DPAPI or possible plaintext fallback), invalidation, multi-account mapping and refresh after login/client update.
6. Trace SQLCipher format inference, page-one HMAC, page decryption, WAL and snapshot lifecycle, corruption and checkpoint retry.
7. Trace database schemas, message shard merge, sender resolution, stable event IDs, attachments, message edits/recalls, timestamp semantics.
8. Trace listener: polling cadence, cursor/watermark, duplicate suppression, restart replay, data races, error reporting.
9. Produce source map from each claim to functions and exact source lines; enumerate tests supporting each mechanism.

**A1 acceptance:** one reproducible end-to-end sequence diagram with function-level references; every component marked `verified in code`, `claimed by README`, or `needs runtime test`.

## Stage A2: OS-independent contracts (deliver `docs/portability-matrix.md`)
- `ProcessDiscovery`: process identity, account binding, binary/version information.
- `MemoryReader`: read eligible mapped pages and distinguish `permission denied` from `no candidate`.
- `KeyCandidateProvider`: candidate metadata only; no raw key in logs.
- `KeyValidator`: validate candidate against target DB page one.
- `SecretStore`: per-user encryption/permissions, key rotation, never sync private secrets.
- `SnapshotProvider`: consistent DB/WAL snapshot or explicit retry/failure.
- `MessageRepository`: normalized read-only cross-shard messages.
- `IncrementalSource`: events with at-least-once delivery and persistent checkpoint.

Portability matrix columns: Windows source / Linux counterpart / reusable algorithm / OS-specific code / version-specific assumptions / test evidence / risk.

## Stage B1: Linux gap analysis (deliver `docs/linux-gap-analysis.md`)
- Pin chatlog-keeper commit and inspect `linux_key.py`, `_linux_process_memory.py`, `linux_memory_scan.c`, `active_key.py`, `_snapshot.py`, `_wal.py`, relevant message parsers.
- Verify actual passive/active control-flow and when `ptrace` or `LD_PRELOAD` can be used. Do not classify `PTRACE_PEEKDATA` fallback as purely passive.
- Measure feature overlap with Windows: key validation, snapshot, parser, multi-shard handling, structured export, stream API.
- Identify version-specific x86_64 memory layout and distinguish Windows MSVC objects from Linux C++ ABI structures.

## Stage B2: Safe feasibility lab (deliver `docs/linux-poc-results.md`)
- Start with a self-written fake process containing synthetic secrets to validate Linux process permissions, parent-child cases and error handling; do not modify system-wide Yama settings or run as root to bypass restrictions.
- Test snapshot consistency and SQLCipher/HMAC algorithms on synthetic fixture DBs.
- Only then, on an account and host the operator owns and consents to test, assess compatibility of official native Linux WeChat with a strictly bounded, read-only-only approach. Log only redacted status, never raw key, private messages or unfiltered memory dumps.
- Distinguish three outcomes: access denied / memory accessible but key undiscoverable / key candidate found and DB HMAC validated.
- No hooks, injections, process patching, anti-detection or protocol imitation in the first PoC. If those prove necessary, stop and revisit risk and authorization.

**B2 gate:** Validated current target version and database with no process writes, injection or debugger attach in the selected read-only workflow. Otherwise do not claim success; consider user-initiated import/export fallback.

## Stage B3: Linux read-only incremental connector
- Prefer chatlog-keeper's existing snapshot and parser foundations; use inotify merely as a trigger, not as the source of truth.
- Re-query shard-aware message identity and persist per-account/per-shard checkpoints; dedupe at-least-once delivery.
- Define event payload with source account, conversation identifier, message stable ID (including shard), sender, timestamp, content type and sanitized content; never leak secrets.
- Recover from WAL checkpoint, app restart, new shards, delayed media writes, deleted/recalled messages, crash between read and checkpoint.
- Local socket or loopback HTTP + user permissions; default disabled cloud upload and external network exposure.

## Stage B4: Stability and deployment
- Test across client restarts, updates, multi-account windows, app logout, rotated keys, busy WAL, large history and container permission boundaries.
- Only after native Linux desktop proves stable evaluate headless GUI and per-user cloud isolation. Do not assume one Linux container per account works.
- Assess platform terms, privacy/consent for counterparties, account restrictions, and licenses of reused projects.

## Verification checklist
- [ ] Source revision pinned
- [ ] Windows key paths mapped independently
- [ ] No write/Hook path in Linux read-only MVP
- [ ] Linux parent-child permissions measured, not assumed
- [ ] Key candidate validated by DB HMAC
- [ ] Multi-shard IDs and sender mappings correct
- [ ] WAL snapshot replay/retry tests pass
- [ ] Crash/restart sends no duplicate externally visible events
- [ ] Keys encrypted/restricted and never logged
- [ ] Account risk documented as unknown, not zero
- [ ] Licensing and client policy reviewed

## Recommended working directory
```
wechat-personal-agent/
  docs/research/windows-mechanism.md
  docs/research/portability-matrix.md
  docs/research/linux-gap-analysis.md
  docs/research/linux-poc-results.md
  docs/architecture/linux-connector.md
  tests/fixtures/        # synthetic only
  linux-connector/       # only after A1/A2 gate
```

## Initial decision
Research and reuse `chatlog-keeper` as the Linux baseline, but do not immediately fork or write production code. First resolve whether current Linux official WeChat exposes reusable, validated key material to a strictly read-only memory reader; if not, the selected constraint fails and the architecture must be reconsidered.
