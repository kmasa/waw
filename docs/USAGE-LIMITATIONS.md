# Submission behavior and limitations

[README](../README.md) · [日本語](USAGE-LIMITATIONS.ja.md)

## What `done` submits

`waw done` submits to the main `waw`, even from a nested child. Eligible repositories are directly under `/waw`, exist in the main environment, and have a recorded starting ref for the child.

Each submission squashes the child's eligible diff per repository. Repeat submissions use the last successful submission as their baseline. Repositories added only in a child are not automatically imported, nor are unsubmitted ancestor changes. Changes that apply without conflicts still need integration testing.

## Conflicts and partial failures

Dirty target repositories and in-progress Git operations block submission. Conflicts remain in the child's submission branch for manual resolution. Resolve the files, stage them, finish the cherry-pick with `git cherry-pick --continue`, then run `waw done` again. Do not mix submission preparation with new development on the original branch.

Final application to the main repository is fast-forward only. There is no automatic reset, stash, or rollback. Multi-repository submissions are not atomic: inspect diagnostics and repository state before retrying a partial failure.

## Retention and deletion

`waw done --keep` requests retention after submission. `done` also retains environments with descendants, mount dependencies, or excluded repositories. Otherwise, child-local files outside Git—including ignored build output—may be deleted with the container.

`waw drop` explicitly discards unsubmitted commits and uncommitted work without confirmation, while still checking ownership and dependencies. Keep backups of valuable data. Plain `exit` leaves the environment available for re-entry with `waw bash <ID>`.

`waw temp` attempts cleanup after completion, failure, and handled interruptions, but cannot guarantee it after SIGKILL or host shutdown. Inspect remaining environments with `waw list`. Shared or external writes are never rolled back by temporary cleanup.

## Copies, shared state, and host integration

- RAM and running processes are not copied. Live disk copies are only crash-consistent; databases and other stateful services may need application-level quiescing. Start required processes in the child.
- `/root` tools and caches are shared. New children can read the main environment's `/waw` through `/waw/waw-root`. This is not a security sandbox for untrusted code. `incus-admin` is effectively host-root access.
- Replacing the main container does not automatically migrate existing mounts.
- Terminal behavior and PowerShell named arguments/native exit-code propagation have known gaps.
- Copy-on-write saves initial duplication, not the CPU, RAM, I/O, or growing disk usage of parallel jobs.

## Verification scope

Existing development environments have recorded fork → edit/test → commit → done cycles, Windows UNC editing, and PowerShell round trips. Dedicated Ubuntu tests cover nested submission, child-side conflict resolution, repeat submission, and temporary-environment operations:

Historical detailed records remain in the internal development repository. See the public [existing-install verification](VERIFICATION.md) for this candidate's checks, including automatic deletion after plain `done`.

This is not fresh-machine validation. Bootstrap is mock-tested only. Multi-repository partial-failure recovery and `done --keep` still need their documented end-to-end Incus checks.

The internal development repository provides `ria check` and `ria test`; its root Riafile and tests are not included in this distribution. These check syntax/diffs and local behavior, including mocks; they do not replace Windows/Incus integration tests.
