# ADR-0010: Output publication, control state and overwrite protection

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Security reviewer |
| Phase | P0.7 (proof) |
| Related | PRD §16.4, §17.2; technical-spec §11.1; ADR-0004, ADR-0008 |

## Context

Builds replace a directory of three artifacts. The technical specification places lock, staging, backup and journal state in a sibling directory `<parent>/.<name>.flowframe/`, which contradicted the PRD rule that generated paths must not escape the output directory. Separately, `compile --output` and `render --output` replaced any file, including source files.

## Alternatives considered

- **Control state inside the target directory.** The target itself is renamed during publication, so the lock and journal cannot live inside it.
- **Control state in a user cache directory keyed by the canonical target path.** Invisible to users and CI caches; cleanup and permission issues on shared runners.
- **One deterministic sibling control directory, explicitly allowed by the PRD.** Adopted.

## Decision

1. For target `<parent>/<name>`, FlowFrame writes only the target and exactly one private control directory `<parent>/.<name>.flowframe/`. The PRD rule "generated paths MUST NOT escape the output directory" is defined to permit this one control directory. The state machine is specified in technical-spec §11.1.
2. `--output-dir` is rejected (policy, exit 5) when it is: a filesystem root; the process working directory; the resolved project root; an ancestor of any resolved source, configuration, theme or asset file; inside a theme directory; or a symlink.
3. Supported filesystems are local POSIX filesystems (APFS, HFS+, ext4, xfs, btrfs, tmpfs). Network filesystems (NFS, SMB) are unsupported; v0.1 does not detect them.
4. A previous output is recognized as FlowFrame-owned when its manifest validates against any manifest version the running release can read (ADR-0004) and its file inventory and hashes match.
5. **Single-file overwrite protection** for `compile --output` and `render --output`:
   - the output path must not resolve to any input file of the invocation;
   - an existing file is replaced only if it is a FlowFrame artifact of the same type: a D2 file with a valid FlowFrame generated header, or an SVG whose first node is the FlowFrame marker comment (ADR-0008);
   - violations are policy errors (exit 5) and leave the file unchanged.
6. Consumers commit or link the final files; `.<name>.flowframe/` is untracked build state.

## Consequences

- Users see one hidden directory next to each output directory and must ignore it in VCS (documented in P8.3).
- Accidental `--output-dir .` or `--output system-model.yaml` cannot destroy sources.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Journal format and fsync points | P0.7 | Crash-window experiments on APFS and ext4. |

## Verification

- P0.7 spike tests both rename crash windows, lock contention and cleanup on APFS and ext4.
- P3.7/P3.8 contract tests for every forbidden output directory and for single-file overwrite refusal.
