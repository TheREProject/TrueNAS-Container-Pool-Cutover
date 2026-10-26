# TrueNAS Container Pool Cutover: Migration, Rollback, and Replication Root-Cause Fix

## Overview

This is a write-up of migrating a self-hosted container platform's storage backend from a 500GB Proxmox vDisk to a 64GB vDisk — and of the first attempt failing, being rolled back, and the actual root cause turning out to be a TrueNAS replication-task behavior that isn't obvious from the naming or configuration of the snapshots involved.

It's also the origin of a separate issue documented in [`Loki Storage and ZFS Snapshot Troubleshooting`](./Loki-Storage-and-ZFS-Snapshot-Troubleshooting.md) — the leftover ZFS snapshots from this same replication run were what later showed up as an apparent Grafana Loki storage/retention problem. That connection wasn't obvious until this write-up and that one were compared side by side.

## Environment

- TrueNAS
- Proxmox (vDisk hypervisor)
- ZFS storage pool, replication factor 1
- Self-hosted container platform, 8 containers total (including Grafana Loki and NextApp)
- Original vDisk: 500GB
- Target vDisk: 64GB

Container host paths (must remain identical across the cutover):

```text
/mnt/ContainerPool/Loki
/mnt/ContainerPool/NextApp
/mnt/ContainerPool/...
```

## Objective

Replace the 500GB vDisk backing the container pool with a much smaller 64GB vDisk, without changing any container-facing paths, and without touching the original disk until the new one was proven reliable — so a bad cutover could be undone without any data loss.

## The Plan

The new, smaller pool (`ContainerPoolNew`) had already been replicated from the original pool and smoke-tested with two containers before this cutover was attempted. The planned sequence was:

1. Stop every container using `/mnt/ContainerPool`.
2. Run the existing periodic snapshot task on the source pool, then run the configured replication task from `ContainerPool` → `ContainerPoolNew`.
3. Verify the replicated datasets and confirm all expected containers' data is present.
4. Export (not destroy) the original pool, keeping the 500GB vDisk intact as a rollback option.
5. Positively identify the new pool by ZFS pool GUID / disk serial — not by pool name, since the old and new pools could both be named `ContainerPool` at different points in the process. Import the correct one under the `ContainerPool` name.
6. Verify the live pool, confirm the container host paths exist exactly as before, and start the containers.
7. Leave the original 500GB vDisk connected but unused, as a rollback path, until the new pool had been proven under real use.

The GUID-based identification step was a deliberate safeguard: once the new pool is imported as `ContainerPool`, it's trivial to import the *wrong* disk under that name if you're going by name alone. Verifying by GUID/serial first removes that risk entirely.

## What Went Wrong

The first cutover attempt did not go cleanly. After importing the new pool and starting the containers, not all of the expected data was present on the new vDisk — some of it simply wasn't there.

Because the original 500GB vDisk had deliberately been left untouched, the rollback path documented in the plan was still available and was used immediately: stop the containers, export the new pool, reconnect and import the original 500GB pool, verify the original `/mnt/ContainerPool/...` paths, and restart the containers. No data was lost — the rollback plan existed specifically so this situation would be recoverable, and it worked as intended.

## Root Cause Investigation

The question was why the replication task hadn't carried over all of the data, given that it had reported success and the earlier two-container test had worked.

The answer turned out to be a specific behavior of TrueNAS's replication tasks: **a replication task only replicates snapshots that were created by its own paired periodic snapshot task — not any snapshot present on the source pool, regardless of that snapshot's naming or whether it matches the replication task's configured name pattern.**

In other words, if the source pool's most current state exists only in a snapshot that wasn't produced by the specific periodic snapshot task feeding that replication task, the replication task will not pick it up — even if the snapshot exists, is recent, and looks like it should match.

## The Fix

The fix was to stop relying on ad-hoc or differently-sourced snapshots and instead:

1. Run the periodic snapshot task that is actually paired with the replication task, immediately before replicating, so the most current state of the pool is captured in a snapshot the replication task will actually recognize.
2. Confirm the new snapshot exists on the source:
   ```bash
   zfs list -t snapshot -r -o name,creation,used,refer ContainerPool
   ```
3. Run the replication task, then confirm the corresponding snapshot landed on the destination:
   ```bash
   zfs list -t snapshot -r -o name,creation,used,refer ContainerPoolNew
   ```

## Re-Execution and Verification

With the correct snapshot now feeding the replication task, the cutover was re-executed following the same plan as before: verify the pool GUID before importing, import the new pool as `ContainerPool`, confirm the live pool and host paths, and start the containers.

This time, verification was done manually and exhaustively rather than by spot-check: every one of the 8 containers was checked individually to confirm its data was present and correct, and that the application itself came up and functioned normally. It wasn't the fastest way to verify, but for a one-time cutover across a small number of containers, thoroughness mattered more than speed.

## Result

- The container platform now runs on the 64GB vDisk instead of the 500GB one — a meaningful storage reclaim.
- All 8 containers' data was confirmed present and correct after the corrected cutover.
- The original 500GB vDisk was kept as a rollback option until the new pool had been running reliably, then retired.
- The root cause (replication tasks only pick up snapshots from their own paired periodic snapshot task) is now a known constraint for any future TrueNAS replication work in this environment.
- A side effect of this same replication run — old snapshots inherited onto the new pool — later showed up as an apparent Loki storage problem, documented separately.

## Lessons Learned

### 1. A successful replication task report isn't the same as complete data

TrueNAS reporting a replication task as "successful" only means the snapshots it selected replicated successfully — not that those snapshots represent the pool's full current state. If the wrong (or an outdated) snapshot fed the task, "success" and "incomplete data" can both be true at once.

### 2. Replication tasks are pickier about snapshot origin than they appear

A snapshot's name or timestamp doesn't tell you whether a given replication task will use it. Only snapshots produced by that task's own paired periodic snapshot task are eligible, in this environment's configuration.

### 3. A rollback plan is only real once it's been used

Writing a rollback procedure is easy. This one was written in advance and then actually exercised under a real, non-hypothetical failure — which is the only way to know whether a rollback plan really works.

### 4. Positive identification beats naming conventions

Verifying the new pool by ZFS GUID / disk serial before importing it — rather than trusting that "the pool named `ContainerPool`" is the right one — is a small step that prevents a genuinely bad outcome (importing the wrong disk under the production name).

### 5. Migrations can have side effects that surface later, elsewhere

The leftover snapshots from this replication didn't cause a visible problem until days later, and in a completely different part of the stack (Loki's apparent storage usage). Worth checking `zfs list -o usedbysnapshots` after any pool replication, not just immediately after the cutover.

## Useful Commands

```bash
# Confirm the source pool's most recent snapshot before replicating
zfs list -t snapshot -r -o name,creation,used,refer ContainerPool

# Confirm the snapshot landed on the destination after replication
zfs list -t snapshot -r -o name,creation,used,refer ContainerPoolNew

# List pools available for import, to find the correct GUID
zpool import

# Import a specific pool by GUID under a given name
zpool import <POOL_GUID> ContainerPool

# Verify the live pool and its datasets
zpool status ContainerPool
zfs list -r ContainerPool

# Confirm host paths match what containers expect
ls -lah /mnt/ContainerPool
```

## Infrastructure / IT Skills Demonstrated

- ZFS pool replication and dataset administration
- TrueNAS storage migration and pool import/export
- Production-style cutover planning with a tested rollback path
- Positive resource identification (GUID/serial) as a safety practice, rather than relying on naming conventions
- Root-cause diagnosis of a platform-specific tool behavior, rather than retrying blindly
- Docker/container platform verification across multiple services
- Evidence-based, closed-loop troubleshooting under a real (not simulated) failure
