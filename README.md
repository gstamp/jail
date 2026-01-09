# jail

A Docker sandbox script for running [opencode](https://opencode.ai) safely in an isolated container.

## Overview

`jail` runs opencode inside a Docker container, providing:

- **Isolation**: Your system is protected from potentially destructive commands
- **Permissive sandbox**: Commands run without confirmation prompts inside the container
- **Project mounting**: Your current directory is mounted at `/workspace`
- **Session persistence**: Your opencode config and conversation history persist across sessions

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running

## Installation

Create a symlink to add `jail` to your PATH:

```bash
# Using /usr/local/bin (requires sudo)
sudo ln -s /path/to/jail/jail /usr/local/bin/jail

# Or using ~/.local/bin (no sudo needed)
mkdir -p ~/.local/bin
ln -s /path/to/jail/jail ~/.local/bin/jail
```

If using `~/.local/bin`, ensure it's in your PATH by adding to `~/.zshrc` or `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## Usage

### Initializing a Project

For a new project, run `--init` to set up the Dockerfile:

```bash
cd /path/to/your/project
jail --init
```

This creates a `Dockerfile.jail` in your project directory that you can customize with your project's dependencies.

### Running the Sandbox

Once initialized (or on first run), start the sandbox:

```bash
jail
```

This starts opencode in the sandbox by default.

### Running Other Commands

You can run any command inside the sandbox:

```bash
# Start an interactive shell
jail bash

# Run a specific command
jail npm install
jail make test

# Run opencode with arguments
jail opencode --help
jail opencode "fix the bug in main.go"
```

### Web UI

Start the opencode web interface with `jail web`:

```bash
# Start web UI on localhost:7500 (foreground, Ctrl+C to stop)
jail web

# Start on a specific port
jail web --port 8080

# Expose to network (not just localhost)
jail web --public

# Run in background (detached mode)
jail web --detached

# Combine options
jail web --public --detached --port 9000

# Pass arguments to opencode web
jail web -- --print-logs --log-level DEBUG
```

**Port Selection**: The web server prefers port 7500 by default. If that port is busy (e.g., another `jail web` instance is running), it automatically tries the next port until it finds a free one.

**Multiple Instances**: You can run multiple `jail web` instances simultaneously for different projects. Each instance gets a unique container name based on the project directory and port.

**Foreground vs Detached**:
- **Foreground (default)**: The terminal stays attached to the container. Press Ctrl+C to stop.
- **Detached (`--detached`)**: The container runs in the background. Use `docker stop <container-name>` to stop it.

## How It Works

1. **First run in a project**: If no `Dockerfile.jail` exists in the current directory, the default one is copied there
2. **Image building**: A Docker image (`opencode-jail`) is built from `Dockerfile.jail`
3. **User identity**: The container runs as your host user (UID/GID), ensuring proper file ownership and Git compatibility
4. **Config setup**: Your `~/.config/opencode` is mounted directly (session history persists!) with a permissive permissions override via `OPENCODE_CONFIG`
5. **Git config**: Your `~/.gitconfig` is mounted read-only for access to global git settings (user name, email, aliases, etc.)
6. **Container execution**: opencode runs with your project mounted at `/workspace`

**Session Persistence**: Your opencode conversation history is stored in `~/.local/share/opencode` and persists across jail sessions. Your settings in `~/.config/opencode` are also preserved. The permissive permissions are applied via OpenCode's [config merging](https://opencode.ai/docs/config/) feature without modifying your original config.

**Git Compatibility**: The container runs as your host user (matching UID/GID), so Git operates normally without "dubious ownership" errors. If you still encounter Git ownership issues, you can use the workaround:
```bash
git -c safe.directory=/workspace add .
```

**mgrep Support**: If [mgrep](https://github.com/mixedbread-ai/mgrep) (semantic grep) is initialized on your host system (`~/.mgrep` exists), the jail automatically mounts your mgrep authentication and configuration. To set up mgrep:

1. Install mgrep on your host: `npm install -g @mixedbread-ai/mgrep`
2. Log in: `mgrep login`
3. The jail will automatically detect and mount your credentials

Alternatively, set the `MXBAI_API_KEY` environment variable to authenticate without browser login.

## Customizing the Dockerfile

The `Dockerfile.jail` in your project directory can be customized to include project-specific dependencies:

```dockerfile
FROM ubuntu:latest

# Install essential tools
RUN apt-get update && \
    apt-get install -y \
    git \
    curl \
    ca-certificates \
    bash \
    && rm -rf /var/lib/apt/lists/*

# Add your project dependencies here, e.g.:
# RUN apt-get update && apt-get install -y nodejs npm

# Install opencode via curl script
RUN curl -fsSL https://opencode.ai/install | bash

# Add opencode to PATH
ENV PATH="/root/.opencode/bin:$PATH"

WORKDIR /workspace
CMD ["/bin/bash"]
```

After modifying `Dockerfile.jail`, rebuild the image by running `jail` again (it auto-rebuilds when the Dockerfile changes).

### Cleaning Up Images

To remove all jail Docker images (useful for freeing disk space or forcing a fresh rebuild):

```bash
jail --cleanup
```

This will list all `opencode-jail-*` images and prompt for confirmation before removing them.

## Environment Variables

The following environment variables are passed through to the container if set:

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | API key for OpenRouter |
| `OPENROUTER_MODEL` | Model to use with OpenRouter |
| `JIRA_API_TOKEN` | Jira API token for integrations |
| `MXBAI_API_KEY` | Mixedbread API key for mgrep (semantic grep) |

## Mounted Files

The following host files are mounted into the sandbox container:

| Host Path | Container Path | Access | Purpose |
|-----------|----------------|--------|---------|
| `~/` (current project) | `/workspace` | Read/Write | Your project files |
| `~/.config/opencode` | `/home/jail/.config/opencode` | Read/Write | opencode configuration & session history |
| `~/.local/share/opencode` | `/home/jail/.local/share/opencode` | Read/Write | opencode conversation history |
| `~/.gitconfig` | `/home/jail/.gitconfig` | Read-Only | Global git configuration (user, email, aliases) |
| `jail-permissions.json` | `/tmp/jail-permissions.json` | Read-Only | Permissive permissions override |
| `~/.mgrep` | `/home/jail/.mgrep` | Read/Write | mgrep authentication (if exists) |
| `~/.config/mgrep` | `/home/jail/.config/mgrep` | Read-Only | mgrep configuration (if exists) |

## License

MIT

