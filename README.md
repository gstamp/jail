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

## How It Works

1. **First run in a project**: If no `Dockerfile.jail` exists in the current directory, the default one is copied there
2. **Image building**: A Docker image (`opencode-jail`) is built from `Dockerfile.jail`
3. **Config setup**: Your `~/.config/opencode` is mounted directly (session history persists!) with a permissive permissions override via `OPENCODE_CONFIG`
4. **Git config**: Your `~/.gitconfig` is mounted read-only for access to global git settings (user name, email, aliases, etc.)
5. **Container execution**: opencode runs with your project mounted at `/workspace`

**Session Persistence**: Your opencode conversation history is stored in `~/.local/share/opencode` and persists across jail sessions. Your settings in `~/.config/opencode` are also preserved. The permissive permissions are applied via OpenCode's [config merging](https://opencode.ai/docs/config/) feature without modifying your original config.

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

After modifying `Dockerfile.jail`, rebuild the image:

```bash
docker rmi opencode-jail
jail
```

## Environment Variables

The following environment variables are passed through to the container if set:

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | API key for OpenRouter |
| `OPENROUTER_MODEL` | Model to use with OpenRouter |
| `JIRA_API_TOKEN` | Jira API token for integrations |

## Mounted Files

The following host files are mounted into the sandbox container:

| Host Path | Container Path | Access | Purpose |
|-----------|----------------|--------|---------|
| `~/` (current project) | `/workspace` | Read/Write | Your project files |
| `~/.config/opencode` | `/root/.config/opencode` | Read/Write | opencode configuration & session history |
| `~/.local/share/opencode` | `/root/.local/share/opencode` | Read/Write | opencode conversation history |
| `~/.gitconfig` | `/root/.gitconfig` | Read-Only | Global git configuration (user, email, aliases) |
| `jail-permissions.json` | `/tmp/jail-permissions.json` | Read-Only | Permissive permissions override |

## License

MIT

