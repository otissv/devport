# AGENTS.md

## Project overview

Devport is a lightweight, framework-agnostic Linux CLI that assigns persistent TCP ports to development projects and injects the assigned port into a command's environment. Its purpose is to prevent accidental port changes and reduce conflicts between local development services.

## Architecture and constraints

- Keep the executable `devport` as a standalone Python 3.7+ script using only the standard library and Linux system utilities.
- Support Fish, Bash and other shells equally; avoid shell-specific execution logic.
- Use `argparse` for commands, `ss -H -ltnp` for TCP listener discovery and `/proc/<pid>/cwd` for process directory inspection.
- Do not add framework detection, a daemon, a database, project-local configuration files or dependencies without explicit approval.
- Prefer simple functions and direct control flow over unnecessary abstractions.
- Preserve backwards compatibility with existing commands and registry data.

## CLI contract

| Command | Behaviour |
| --- | --- |
| `devport run <command> [args...]` | Automatically allocate or reuse a port and execute the supplied command. |
| `devport list` | Show registered projects, ports, status and paths. |
| `devport scan` | Show active TCP listeners, PIDs and working directories. The current implementation also shows `RESERVED FOR`. |
| `devport set <port>` | Assign or change the current project's port. |
| `devport kill [port]` | Send SIGTERM to confirmed project listeners; default to the current project. |
| `devport kill --force <port>` | Explicitly permit termination of unrelated or unregistered listeners. |
| `devport remove [port]` | Release an inactive allocation; default to the current project. |

Registration is implicit on `run` or `set`; do not introduce a separate mandatory registration step.

## Registry and allocation invariants

- Store allocations at `$XDG_CONFIG_HOME/devport/registry.json`, defaulting to `~/.config/devport/registry.json`.
- Use the resolved absolute invocation directory as the project identity. The schema is a JSON object mapping paths to integer ports, e.g. `{ "/home/user/projects/cms": 3000 }`.
- Allocate automatically from 3000–9999 inclusive; allow explicit ports from 1–65535 subject to availability and OS permissions.
- Never assign a port reserved by another project or currently unavailable for binding.
- Reuse existing allocations and fail rather than silently choose a different port when a reservation is occupied.
- Preserve registry locking with `fcntl.flock` and atomic writes using a temporary file and `os.replace`.
- Reject a port change or removal while the existing port is listening. Setting the same port is a no-op.
- Killing a server must not release its reservation.
- Preserve allocations across process exits and reboots; do not automatically remove missing project directories.
- Do not claim registry reservations are OS-level locks: a separate process can bind a port after the availability check.

## Execution and process safety

- Accept arbitrary executables and arguments, not just Node.js or pnpm.
- Preserve the parent environment and override only `PORT` with the allocated port unless explicitly requested otherwise.
- Use argument arrays and `os.execvpe`; never concatenate user input into a shell command or use `shell=True`.
- Do not hold the registry lock for the lifetime of a launched server.
- Do not rewrite framework configuration, application URLs or CORS settings. Applications are responsible for respecting `PORT` and disabling automatic port fallback (e.g. Vite `strictPort: true`).
- Identify listener ownership using the visible process working directory, with directory-boundary-aware matching. Treat unavailable PID or directory information as unknown, not as proof of ownership.
- Default to SIGTERM, not SIGKILL. Refuse unrelated or unconfirmed listeners unless the user explicitly supplies `--force`; do not bypass OS permissions.
- Do not claim a server stopped merely because a signal was sent; supervisors may restart it.
- Remember that `pnpm --dir` does not change the registry identity; separate services need separate invocation directories.

## Output and error handling

- Keep output concise, readable and suitable for a terminal.
- `list` distinguishes `Running` (ownership confirmed), `Occupied` (listener ownership unconfirmed) and `Stopped` (no listener found).
- `scan` reports listening TCP ports in the current network namespace; show unknown process information as `?`.
- Prefer minimal table columns. Removing the existing `RESERVED FOR` scan column is a possible simplification, not an implemented change.
- Report actionable errors to stderr and return nonzero exit codes for failures.
- Avoid silent failures and destructive recovery from invalid registry data.

## Development and verification

Run basic checks after modifying the script:

```sh
python3 -m py_compile devport
./devport --help
```

Use temporary project directories and an isolated `XDG_CONFIG_HOME` for tests. Never modify the user's real registry or kill unrelated processes during testing. Prefer standard-library `unittest` if introducing automated tests.

Verify relevant behaviour when changing the CLI:

- First-run allocation, reuse across invocations and different allocations for different directories.
- Argument boundaries, environment preservation, `PORT` injection and child exit-code propagation.
- Occupied and reserved port rejection, concurrent registry operations and persistence.
- `set` validation, no-op behaviour and rejection while an existing server listens.
- `list` and `scan` output, including unknown process metadata.
- `kill` ownership checks, explicit force handling and preservation of reservations.
- `remove` rejection for active listeners and successful release of inactive allocations.

Use controlled test-owned listener processes for integration tests and clean them up afterward. Report which checks were actually run and any platform limitations or untested cases.

## Documentation and change scope

- Update `README.md` whenever commands, flags, defaults, output, installation instructions, registry behaviour or limitations change.
- Keep usage examples consistent with the executable and use shell-neutral syntax where possible.
- Preserve existing functionality unrelated to the requested change; do not introduce speculative features or broad refactors.
- Do not commit, push, publish or install changes without an explicit request.
- Keep this file focused on durable project rules; update it when architectural decisions or invariants change.
