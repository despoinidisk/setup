# setup

A reusable script for managing project services (start, stop, logs). The `setup` script is identical across all projects — only the `.setup` hook file is project-specific.

## Usage

```
setup up [-d|--detach]   Start all services
setup down               Stop all services
setup logs               Tail service logs (detached mode only)
```

### Examples

```bash
# Start services in the foreground (Ctrl+C to stop)
setup up

# Start services in the background
setup up -d

# View logs from background processes
setup logs

# Stop all services
setup down
```

## Install

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/despoinidisk/setup/refs/heads/main/install)
```

## How It Works

1. Copy the `setup` script into your project root.
2. Create a `.setup` file defining `up()` and `down()` functions.
3. Run `setup up` from the project root.

The script sources `.setup` from the current working directory and delegates to the hooks you define.

## The `.setup` File

Create a `.setup` file in your project root with two functions:

```bash
up() {
    local detached="${1:-0}"
    cd "$ROOT"

    _start_bg api "$detached" make api.start
    _wait_http "http://localhost:8000/health" "API"

    _start_bg web "$detached" npm run dev
    _wait_http "http://localhost:3000" "Frontend"

    INFO "All services started."
}

down() {
    _kill_component api
    _kill_component web
}
```

### Available Utilities

The following are provided by `setup` and available inside `.setup`:

#### Logging

| Function | Description |
|----------|-------------|
| `INFO "msg"` | Log to stdout |
| `WARN "msg"` | Log to stderr |
| `ERRO "msg"` | Log to stderr |

#### Process Management

| Function | Description |
|----------|-------------|
| `_start_bg NAME DETACHED CMD [ARGS...]` | Start a command in the background and track its PID. If `DETACHED` is `1`, output is redirected to `$LOGS_DIR/NAME.log`. |
| `_kill_component NAME` | Stop a tracked process and all its children by component name. |
| `_save_pid NAME PID` | Manually save a PID file for a named component. |
| `_wait_http URL LABEL [ATTEMPTS]` | Poll an HTTP endpoint every second until it responds. Default timeout: 90 attempts. |

#### Environment

| Function | Description |
|----------|-------------|
| `_load_env` | Source `.env` from the project root, exporting all variables. |

#### Paths

| Variable | Description |
|----------|-------------|
| `ROOT` | Project root (`$PWD`) |
| `PIDS_DIR` | PID files directory (`.setup-pids/`) |
| `LOGS_DIR` | Log files directory (`.setup-logs/`) |

## Directory Structure

```
project/
  setup          # The reusable script (identical across projects)
  .setup         # Project-specific hooks (up, down)
  .env           # Optional environment variables (auto-loaded by _load_env)
  .setup-pids/   # Created automatically — PID files for tracked processes
  .setup-logs/   # Created automatically — logs from detached processes
```

## Integration With Git

Add the following to your `.gitignore`:

```
.setup-pids/
.setup-logs/
```

The `setup` script itself can be added to your project via symlink, copy, or as a git submodule. Only `.setup` should be committed per project.
