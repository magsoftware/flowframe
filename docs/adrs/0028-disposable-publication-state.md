# ADR-0028: Output publication with disposable control state

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Tech lead, Security reviewer, Product owner |
| Phase | P0.1 (question), P0.7 (proof) |
| Related | PRD §16.4, §17.2, §22 (AC 19); technical-spec §11.1, §18; ADR-0010 |

## Context

ADR-0010 and technical-spec §11.1 define a seven-step state machine: an advisory lock per target, a staging directory, one retained backup, a recovery journal with fsync points, and recovery that restores or completes an interrupted publication on the next run. P0.7 has to prove it with injected crashes on APFS and ext4.

The backup and journal protect the previous output, but the previous output is not irreplaceable data. Every FlowFrame output is a deterministic function of source files that the build never modifies (PRD §16.1, ADR-0010 item 2). Its loss is repaired by running the same build again, and committed outputs can also be restored from version control. The parts of ADR-0010 that protect user data are the refusal of forbidden output directories and the refusal to overwrite anything that is not a FlowFrame artifact. Those parts do not depend on the backup or the journal.

## Alternatives considered

- **Keep ADR-0010 as accepted.** Restores the previous output after an interrupted publication, at the cost of the journal, lock, recovery logic, crash-window tests and a P0 spike.
- **Disposable control state: stage, verify, swap, delete; recover by rebuilding.** Proposed here.
- **Replace the three files one by one inside the target.** Leaves a mixed artifact set after a crash, which the ownership check would then reject as modified. Rejected.

## Proposed decision

1. ADR-0010 items 2 (forbidden output directories), 3 (supported filesystems), 4 (ownership recognition), 5 (single-file overwrite protection) and 6 (control directory is untracked) remain in force unchanged.
2. The private directory `<parent>/.<name>.flowframe/` remains the only path outside the target that a build writes. It contains a FlowFrame ownership marker file and holds only disposable state. Any build may delete its contents, but only when the marker is present.
3. A build prepares and verifies the complete artifact set in a fresh staging directory inside the control directory. Validation and render failures leave the target untouched.
4. Publication: if the target exists and passes the ownership check, rename it into the control directory. Then rename staging to the target, and delete the old output. There is no journal, retained backup or restore step.
5. An interrupted publication can leave the target missing, never mixed or partially written. The next build treats a missing target as a first publication and removes leftover control state. The diagnostic for an interrupted or failed publication says to rerun the build. It does not promise that the previous output was restored.
6. Concurrent builds into the same target are unsupported, and no lock is taken. Each build still ends either with a complete artifact set at the target or with a failing filesystem operation reported as an execution error (exit 4). Neither outcome overwrites content that failed the ownership check.

## Consequences

If accepted:

- Removed from v0.1: the advisory lock, the recovery journal and its fsync points, backup retention and restore, and the P0.7 crash-window spike. The P3.8 work package and the publication part of the P6.3 security suite shrink accordingly.
- Changed public behavior: after an interrupted publication, the previous output may be missing until the next build. AC 19 becomes "a render or validation failure never modifies the previous output; an interrupted publication never leaves a mixed artifact set and is repaired by rerunning the build".
- `fcntl` locking is no longer required, which also removes a POSIX-only dependency from publication.
- Documents to change on acceptance: PRD §16.4, §17.1 (execution category: no lock), §17.2, §22 AC 19; README "Planned CLI"; technical-spec §11.1, §18; ADR-0010 marked superseded in part; implementation-plan P0.7, P3.8, P6.3.

## Verification

- Contract tests for every forbidden output directory and for refusing non-artifact content (unchanged from ADR-0010).
- An interruption injected between the two renames leaves either the old target or no target, and the next build publishes a complete set.
- A control directory without the ownership marker is never deleted.
- Two concurrent builds to one target never leave a mixed artifact set. Each either publishes a complete set or fails with exit 4.
