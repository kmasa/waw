# 2026-10-26: existing WSL installation verification

This verifies the public candidate's existing-install path, **not bootstrap on a clean WSL installation**.

## Environment and candidate

- Existing Windows + WSL2 Ubuntu / Incus-managed Btrfs / Ubuntu main container.
- Public runtime files are identical to development revision `dff0afbef9c5`.
- Ran the candidate's `src/wsl/Riafile install` as the non-root WSL user. It completed and updated the WSL and Windows launchers; no systemd/polkit changes were needed in this environment.
- Normal launchers delegate to the existing main container's code. Its `src/` content was checked against the public candidate. This was not a separate, newly bootstrapped installation.

## Results

- Development `ria check` and `ria test` passed. These include mock-based host tests and real local Git tests, not a substitute for the live checks below.
- Windows `waw list` and `waw bash waw` succeeded through the normal Windows launcher.
- Normal `waw new`, `waw fork`, and entry into the child succeeded.
- A dedicated project's file was edited and committed in the child; the main environment remained unchanged before submission.
- Argument-bearing `waw pwsh Write-Output ...` succeeded and returned to the child shell. socat emitted TTY/shutdown warnings; this does not validate all argument/exit-code cases or interactive PowerShell.
- Plain `waw done` integrated the change with a squash commit, returned to the main environment, and automatically deleted the clean child. The main project's content and clean Git state were checked.
- `waw add` cloned a dedicated local file URL successfully. This is not an external Git service availability test.
- `waw temp` inherited the project file, ran its command, and deleted its temporary child. The stdin-driven test's subsequent shell input was consumed, so the test shell was closed separately after confirming completion and cleanup. No custom test PTY was used.
- Existing unrelated containers were not deleted.

## Not verified here

Clean-WSL bootstrap, fresh package/tool installation, shutdown/recovery, all interactive UI operations, error paths, and performance guarantees remain outside this verification. Existing limitations still apply.
