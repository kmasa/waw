# WAW (WSL AI Workspace)

**Fast parallel testing and AI-assisted coding.**

- **Fork in 1 second**
- Clone entire Linux development environments, including `node_modules` and intermediate build files
- No need to rerun `npm install`
- Preserve build artifacts across forks
- Integrate changes with Git squash commits

[日本語](README.ja.md)

WAW is an experimental CLI for **Windows + WSL2** that copies Ubuntu development containers using **Incus and Btrfs**, then integrates committed changes back into a main environment. It is built for trying changes independently—including running AI coding agents on separate tasks—without sharing one working tree.

**Experimental / Proof of Concept.** The core workflow has been used to develop WAW itself. This is working software, not a production-ready environment manager. Installation is experimental and has not yet been verified on a clean WSL2 environment.

**WAW is not an AI orchestrator.** It handles environment copies and Git integration; you choose the agents, assign tasks, and review their work.

## Why fork an environment?

Git branches separate source history, but they do not separate installed packages, build outputs, or local services. WAW copies the container's disk state so each task can start from an existing development setup and work in its own environment.

- **Branch the environment:** Incus copies containers on an Incus-managed Btrfs pool, using copy-on-write storage.
- **Work independently:** edit, build, test, and commit in separate children; fork a child again when needed.
- **Bring back code, not the whole disk:** `waw done` submits eligible Git changes to the main `waw` environment. It does not merge installed packages or databases.
- **Keep Windows tools:** edit the same files through WSL UNC paths, or open Windows PowerShell from a container with `waw pwsh`.

The environments are not completely isolated: `/root`, including development tools and caches, is shared. See [limitations](#limitations-and-safety) before running agents or unfamiliar code.

### The workflow at a glance

After setup, from a clean Git project in the main `waw` environment, with Git author identity configured:

```bash
waw fork --bash
printf 'Hello from a WAW child\n' > hello.txt
git add hello.txt
git commit -m "Add a greeting"
waw done
```

The change returns to the main environment. For real work, edit and test in the child before committing. See [Quick start](#quick-start) for entering WAW and creating a project.

## Parallel AI development

For example, give one agent a feature and another a test task, each in a separate child:

```text
Main environment: waw
  ├─ fork → child A → AI 1: implement a feature → test → commit → done ─┐
  └─ fork → child B → AI 2: improve tests       → test → commit → done ─┤
                                                                     ↓
                                                        Main environment: waw
```

Run `waw fork` twice from the main environment, then use `waw bash <ID>` in separate terminals to enter the children. Start your agent inside each one and review its changes before submitting. `fork` creates the environment; it does not start an agent. Parallel task assignment and review are user-managed, not an autonomous orchestration feature.

Even a child's child submits directly to the main `waw`, not to its immediate source. Git conflicts still require resolution, and independently applicable changes still need integration testing.

## Demo and reference timings

In an existing Windows + WSL2 / Incus / Btrfs setup, we forked a built **ripgrep 14.1.1** environment, ran tests in child A alongside changes/build/tests in child B, integrated B's change, and tested the integrated version. The children inherited a working release binary; source/target writes did not affect the other environments. Shared `/root` and host resources remain shared, and inherited build outputs do **not** guarantee that recompilation is unnecessary.

The **2026-10-25** follow-up used Ubuntu 24.04, Rust/Cargo 1.85.1, jobs=2 / test threads=2, and inherited target/cache. Other agents and editors were running, so these are **reference observations, not quiet-system benchmarks or performance guarantees**.

| Measured interval | Median of 3 runs |
|---|---:|
| Normal fork response | 0.945 s |
| Fork start → child's `rg` response, including Windows transport | 2.661 s |
| Incus copy, separately instrumented backend runs | 0.215 s |
| Incus start, same separate runs | 0.258 s |
| Two children's parallel build → test operation | 14.542 s |

The copy/start measurements are not an additive breakdown of the normal CLI. All 12 follow-up test runs (standalone and parallel) passed 1,106 tests each. Timing boundaries and cache conditions differ, so these results do not establish a parallel speedup.

**Known caveats:** separate probes reproduced intermittent access-time-order failures and confirmed that backward wall-clock jumps can break the tests' file-creation-order assumption. This does not explain every historical failure. Windows entry/exit was checked through an existing launcher, not a newly installed candidate. A complete Windows demo of the final candidate and quiet-system baseline measurements remain unfinished. The original integration demo used the old immediate-parent `done` behavior; it is not validation of today's main-environment submission or automatic deletion.

See the [demo and benchmark record](docs/DEMO-BENCHMARK.md) (Japanese) for exact conditions, per-run values, measurement commands, failures, and remaining work.

## Requirements and installation

The target setup is:

- Windows with **WSL2 Ubuntu**, WSL interop, and **systemd** enabled in WSL.
- A working Incus daemon and a WSL user permitted to manage it.
- Ruby and its standard library in WSL; the generated single-file task runner, `src/tools/ria.dist.rb`, is included in this repository.
- Network access for Ubuntu images, packages, and tool installers. The default container image is `images:ubuntu/24.04`.
- For Windows editing, an editor that supports UNC paths. PowerShell integration uses `pwsh.exe`, falling back to `powershell.exe`.

Native Linux is not a verified target. Windows Ruby is not required for the normal Windows launcher.

### Bootstrap from local files

**This path is implemented and covered by local mock tests, but has not been verified on a fresh WSL installation.** Try it only in a development environment you can inspect and recover. The public repository is [kmasa/waw](https://github.com/kmasa/waw); there is no published one-line installer to recommend.

With a trusted local copy of the repository or the complete WAW distribution in WSL, install Ruby if it is missing:

```bash
sudo apt-get update
sudo apt-get install -y ruby
```

Then, as a **non-root WSL Ubuntu user**, run from the WAW directory:

```bash
ruby --disable-gems ./src/tools/ria.dist.rb --riafile src/wsl/Riafile bootstrap
```

Bootstrap requests approval for host changes, prepares the Btrfs pool, network, shared `/root` volume, and main Ubuntu container, then installs the launchers. The default initial pool size is `8GiB`; pass `size=20GiB` to choose a different initial size. Existing pools are not resized. Guest setup installs missing mise, Node.js, Pi, and Herdr using external installers where applicable.

systemd enablement and Incus access must be arranged separately. Bootstrap does not grant `incus-admin`, configure service startup, or shut down WSL. Later launcher setup also requests approval for systemd mount and polkit configuration. Unknown resources with matching names are rejected rather than overwritten. Failures leave resources and diagnostics in place; there is no automatic rollback.

Read the [installation guide](docs/INSTALL.md) before proceeding. For an already configured WAW main container, launcher setup and host changes are described in the [Windows guide](docs/WINDOWS.md). `install` alone does not create the main container. These detailed guides are currently in Japanese.

## Quick start

After setup, open a new Windows terminal:

```powershell
waw
waw status
waw bash waw
```

`waw` lists the commands available at that entry point. The last command enters the main Ubuntu environment. From its Bash shell, create a small project and fork it:

```bash
waw new waw-example
cd /waw/waw-example
waw fork --bash
```

You are now in a child, at the same project path. With your Git author identity configured, make a change and submit it:

```bash
printf 'Hello from a WAW child\n' > hello.txt
cat hello.txt
# For a real project, run its tests before committing.
git add hello.txt
git commit -m "Add a greeting"
waw done
```

On success, you return to the main `waw` environment and the project contains `hello.txt`. `done` integrates eligible committed changes and deletes the child only when it can safely close it. `waw done --keep` requests that the child remain after submission. Plain `exit` leaves an environment without submitting or deleting it; re-enter with `waw bash <ID>`.

### Commands

| Command | Purpose |
|---|---|
| `waw status` | Read-only host and Incus diagnostics; Windows / WSL only. |
| `waw new <name>` | Create a Git project under `/waw` in the main environment. |
| `waw add <URL>` | Clone a Git repository into the main environment. |
| `waw fork [--bash]` | Copy the current environment; stay in place unless `--bash` is supplied. From Windows / WSL, the source is the main environment. |
| `waw bash [ID]` | Enter an environment by ID, or choose from a list. |
| `waw list` | List owned environments, their state, temporary/retained classification, and source. |
| `waw done [--keep]` | Submit eligible Git changes and return to the main environment; child containers only. |
| `waw drop [ID]` | Delete a child, **discarding unsubmitted commits and uncommitted work without confirmation**. ID is required outside a child. |
| `waw temp <cmd> [args...]` | Fork, run a command at the same path, then discard the temporary child. |
| `waw pwsh [command args...]` | Open Windows PowerShell at the corresponding UNC path; from a container only. |
| `waw herdr` | Request a Herdr tab; requires an available Herdr server. In a container, uses the current directory. |

`new` and `add` always target the main environment, even when called inside a child. Use `fork`, not argument-free `add`, to branch an environment. WAW does not manage Git remotes or push your changes.

For a disposable command, use, for example, `waw temp bash -c 'pwd; git status --short'` inside a project. Only the child's local changes are discarded; shared or external writes are not rolled back.

## Limitations and safety

- **Disk copy, not process cloning.** RAM and running processes are not copied. A live copy is only crash-consistent; databases and other stateful services may need application-level quiescing. Start required processes in the child.
- **Shared state and privileges.** `/root` tools and caches are shared, and new children can read the main environment's `/waw` through `/waw/waw-root`. Incus administration is powerful; `incus-admin` is effectively host-root access. WAW is not a security sandbox for untrusted agents or code.
- **Git-only submission.** `done` submits eligible committed changes, not packages or databases. Conflicts need manual resolution; there is no automatic rollback, and multi-repository submission is not atomic.
- **Deletion is real.** `done` may delete the child, including files outside Git. `drop` discards unsubmitted work without confirmation. Back up valuable data; temporary cleanup is not guaranteed after SIGKILL or host shutdown.
- **Integration remains experimental.** Terminal and PowerShell integration have known gaps. Parallel jobs still share host resources.

See [submission behavior and limitations](docs/USAGE-LIMITATIONS.md) for eligibility, squash/repeat submissions, conflict handling, retention, and cleanup details.

### What has been verified?

Existing development environments have recorded successful fork → edit/test → commit → done cycles, Windows UNC editing, and PowerShell round trips. Dedicated Ubuntu tests also cover direct submission from nested children, child-side conflict resolution, repeat submission, and temporary-environment operations. Historical detailed records are retained in the internal development repository. See the public [existing-install verification](docs/VERIFICATION.md) for the checks performed on this candidate.

This is not fresh-machine validation. Bootstrap is mock-tested only. On 2026-10-26, existing-install setup, the normal Windows workflow, and automatic deletion after plain `done` passed live checks. Multi-repository partial-failure recovery and `done --keep` still need their documented end-to-end Incus checks. Turnkey installation and broader platform support are not current guarantees.

The internal development repository provides `ria check` and `ria test`; its root Riafile and tests are not included in this distribution. They check syntax/diffs and local CLI, storage, bootstrap, and Git behavior, including mocks; they do not replace Windows/Incus integration tests.

## License

[MIT License](LICENSE) — Copyright (c) 2026 kmasa.
