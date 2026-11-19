# devport

A lightweight, framework-agnostic Linux CLI for assigning persistent development server ports to projects. Run any command with a dedicated `PORT` environment variable, without manually tracking port assignments.

## Requirements

- Linux with `/proc` and `ss` (provided by `iproute2` on Arch Linux/CachyOS).
- Python 3.7 or later.
- Fish, Bash or another shell. No Python packages are required.

## Installation

From the cloned repository:

```sh
mkdir -p ~/.local/bin
cp devport ~/.local/bin/devport
chmod +x ~/.local/bin/devport
```

For Fish, add the installation directory to your PATH:

```fish
fish_add_path ~/.local/bin
```

For Bash, add this line to `~/.bashrc` and reopen your terminal:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Verify installation:

```sh
devport --help
```

To update, pull the latest repository changes and repeat the copy command.

## Quick start

Navigate to your project directory and start your server:

```sh
cd ~/projects/my-app
devport run pnpm dev
```

On the first run, devport automatically allocates an unused, unreserved TCP port between 3000 and 9999. Subsequent runs from the same directory reuse that port. The command and all its arguments are passed through to the executable, with `PORT` injected into its environment.

For example, if port 3000 is allocated, the equivalent command is:

```sh
PORT=3000 pnpm dev
```

## Commands

| Command | Description |
| --- | --- |
| `devport run <command> [args...]` | Allocate or reuse a port and run a command. |
| `devport list` | Show registered projects, ports, statuses and paths. |
| `devport scan` | Show listening TCP ports, PIDs, and working directories. |
| `devport kill [port]` | Send SIGTERM to listeners belonging to the registered project. Defaults to the current directory's port. |
| `devport kill --force <port>` | Allow terminating unrelated or unregistered listeners (use with caution). |
| `devport set <port>` | Set or change the current directory’s reserved port. Both the old (if changing) and new ports must not be listening. |
| `devport remove [port]` | Release a registered port. Defaults to the current directory's allocation. The port must not be listening. |

### Examples

```sh
devport run pnpm dev
devport run npm start
devport run node server.js
devport run bun dev
devport run pnpm --dir kanban/web dev
devport list
devport scan
devport kill 3000
devport set 4000
devport remove 4000
```

Projects are identified by the absolute, resolved working directory from which devport is invoked. For separate allocations in a monorepo, change into each application directory before running devport; `pnpm --dir` alone does not change the registry key.

### Set or change a port

From the project directory:

```sh
devport set 4000
devport run pnpm dev
```

`set` updates the existing allocation (releasing the old port), or creates an allocation if the directory is not yet registered. Subsequent runs use the selected port permanently until you change or remove it.

- Explicit ports may be between 1 and 65535, subject to operating-system permissions and availability; automatic allocations remain between 3000 and 9999.
- A port already reserved by another project or unavailable for binding is rejected without changing the registry.
- Stop the current server before changing its allocation (`devport kill` if the listener belongs to the project).
- Setting the same port again is a no-op.
- Update any dependent API URLs or CORS origins yourself when changing ports.

### Example registry listing

```text
PROJECT       PORT  STATUS   PATH
------------  ----  -------  -----------------------------
kanban        3000  Running  /home/user/projects/kanban
cms           3001  Stopped  /home/user/projects/cms
image-editor  3002  Running  /home/user/projects/image-editor
```

`Running` means visible listener PIDs have working directories inside the registered project; `Occupied` means the port is listening but ownership was not confirmed; `Stopped` means no TCP listener was found.

## Application configuration

Devport injects `PORT`; your application must actually use it. It does not detect frameworks, rewrite configuration files or change CORS settings.

### Vite / React / TanStack Start

Add the following to the existing `defineConfig` options in `vite.config.ts`:

```ts
server: {
  port: Number(process.env.PORT ?? 3000),
  strictPort: true,
},
```

Then start the application:

```sh
devport run pnpm dev
# Or, from the parent directory:
devport run pnpm --dir kanban/web dev
```

`strictPort` prevents Vite from silently falling back to another port.

### Next.js

Next.js respects `PORT` when starting the server. Ensure your package script does not override it with an explicit port flag.

```sh
devport run pnpm dev
```

### Node.js / Express

```js
const port = Number(process.env.PORT ?? 3000);
app.listen(port, () => console.log(`Listening on ${port}`));
```

```sh
devport run node server.js
```

If your frontend and backend use different allocations, configure API URLs and allowed CORS origins to match their registered ports.

## Registry

Allocations are stored in `${XDG_CONFIG_HOME:-~/.config}/devport/registry.json` (normally `~/.config/devport/registry.json`). The registry maps absolute project paths to port numbers:

```json
{
  "/home/user/projects/kanban": 3000,
  "/home/user/projects/cms": 3001
}
```

Allocations persist across terminal sessions and reboots. Killing a server does not remove its allocation. If an allocated port is occupied when starting a project, devport fails rather than choosing another one.

## Limitations

- Reservations are maintained by devport, not enforced by the operating system; other programs can still bind to a reserved port, including between the availability check and server startup.
- Scanning is limited to listening TCP sockets in the current network namespace; process details may be hidden by permissions.
- `kill` sends SIGTERM and does not guarantee that a process exits or that a development supervisor will not restart it. It does not release the reservation.
- Moving or renaming a project changes its directory identity; remove the old allocation if it is no longer needed.
- One allocation per invocation directory; use separate directories for different services.
