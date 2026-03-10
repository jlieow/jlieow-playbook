# Coding Agent Sandboxes

Coding agents like Claude Code, Gemini CLI, and Codex CLI run as interactive command-line tools. They read your code, propose changes, and execute shell commands on your behalf. That power is useful, but it means a misconfigured agent (or a bad prompt) can modify files outside the project, install unexpected packages, or leak credentials.

Running the CLI inside a Docker container gives you an isolation boundary: the agent can only see the files you explicitly mount in, and everything is discarded when the container exits.

## How it works

1. **Build** a Docker image that contains the CLI tool and a non-root user.
2. **Run** the container, mounting your project directory and credentials.
3. The agent operates inside the container. When it exits, the container is removed (`--rm`), but changes to mounted volumes persist on the host.

## Structure

```
sandboxes/
  Dockerfile.claude.template    # Minimal Claude Code sandbox
  Dockerfile.codex.template     # Minimal Codex CLI sandbox
  Dockerfile.gemini.template    # Minimal Gemini CLI sandbox
  terraform/                    # Go + Terraform provider dev
    Dockerfile.claude
    Dockerfile.gemini
    readme.md
```

The **templates** at the root are lightweight starting points — just the CLI and a non-root user. Copy one into a new subdirectory and add whatever toolchains your project needs.

Each **subdirectory** is a purpose-built variant with its own readme covering build, run, and domain-specific notes.

| Variant | Directory | Description |
|---------|-----------|-------------|
| Template | `.` | Base images, no language toolchains |
| [Terraform](terraform/readme.md) | `terraform/` | Go + Terraform for provider development |

## Templates

### Claude

#### 1. Build the image

```bash
docker build -f Dockerfile.claude.template -t claude-code:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.claude.template -t claude-code:latest .
```

#### 2. Choose an authentication method

**Option A: Databricks-routed credentials (`settings.json`)**

`~/.claude/settings.json` contains the env block (ANTHROPIC_BASE_URL, ANTHROPIC_AUTH_TOKEN, etc.) that Claude Code reads at startup. It is mounted read-only so the container cannot modify it.

**Option B: Direct API key**

Set `ANTHROPIC_API_KEY` in your shell (e.g. `export ANTHROPIC_API_KEY=sk-...`). The `-e` flag passes it into the container without embedding it in the command.

#### 3. Launch the agent

There are two ways to launch: **headless** (pass a prompt, get a result, exit) and **interactive** (open a REPL inside the container).

**Headless — run with a prompt**

Option A (settings.json):
```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  claude-code:latest --prompt --verbose --dangerously-skip-permissions \
  "your prompt here"
```

Option B (API key):
```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code:latest --prompt --verbose --dangerously-skip-permissions \
  "your prompt here"
```

**Interactive — open a REPL**

Option A (settings.json):
```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  claude-code:latest --verbose --dangerously-skip-permissions
```

Option B (API key):
```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code:latest --verbose --dangerously-skip-permissions
```

#### 4. Create a shell shortcut

Typing a full `docker run` command every time is tedious. Add a shell function to `~/.zshrc` (or `~/.bashrc`) that wraps the command into a single word:

```bash
claudex() {
  docker run -it --rm \
    --workdir $(pwd) \
    -v $(pwd):$(pwd) \
    -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
    claude-code:latest --verbose --dangerously-skip-permissions "$@"
}
```

This mounts the current directory at the same absolute path inside the container and sets `--workdir` to match, so the agent sees the same paths you do on the host.

Reload your shell and run from any project directory:

```bash
source ~/.zshrc

cd ~/your-project
claudex
```

Pass extra arguments as needed, e.g. `claudex --prompt "your prompt here"`.

### Gemini

#### 1. Build the image

```bash
docker build -f Dockerfile.gemini.template -t gemini-cli:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.gemini.template -t gemini-cli:latest .
```

#### 2. Launch the agent

**Option A: Credentials file (`.env`)**

```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.gemini/.env:/home/agent/.gemini/.env:ro \
  gemini-cli:latest --yolo \
  "your prompt here"
```

`~/.gemini/.env` contains the Gemini API credentials. Mounted read-only so the container cannot modify it.

**Option B: Direct API key**

```bash
docker run -it --rm \
  -e GEMINI_API_KEY \
  -v $(pwd):/workspace \
  gemini-cli:latest --yolo \
  "your prompt here"
```

`GEMINI_API_KEY` must be set in your shell (e.g. via `export GEMINI_API_KEY=...`). The `-e` flag passes it through without embedding it in the command.

### Codex

#### 1. Build the image

```bash
docker build -f Dockerfile.codex.template -t codex-cli:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.codex.template -t codex-cli:latest .
```

#### 2. Launch the agent

**Option A: Config file + env var (`config.toml` + `DATABRICKS_TOKEN`)**

`~/.codex/config.toml` contains the provider config (base URL, model, wire API). `DATABRICKS_TOKEN` is the API key referenced by `env_key` in the config. Both are needed at runtime.

```bash
docker run -it --rm \
  -e DATABRICKS_TOKEN \
  -v $(pwd):/workspace \
  -v ~/.codex/config.toml:/home/agent/.codex/config.toml:ro \
  codex-cli:latest \
  "your prompt here"
```

**Option B: Direct OpenAI API key**

```bash
docker run -it --rm \
  -e OPENAI_API_KEY \
  -v $(pwd):/workspace \
  codex-cli:latest \
  "your prompt here"
```

`OPENAI_API_KEY` must be set in your shell (e.g. via `export OPENAI_API_KEY=sk-...`). The `-e` flag passes it through without embedding it in the command.
