# Coding Agent Sandboxes

The safest way to run Coding Agents is in a sandboxed environment. A Docker container isolates the agent from the main system.

## Structure

```
sandboxes/
  Dockerfile.claude.template    # Minimal Claude Code sandbox
  Dockerfile.codex.template     # Minimal Codex CLI sandbox
  Dockerfile.gemini.template    # Minimal Gemini CLI sandbox
  terraform/                    # Go + Terraform for provider dev
    Dockerfile.claude
    Dockerfile.gemini
    readme.md
```

The **templates** at the root are lightweight starting points, just the CLI and a non-root user. Copy one into a new subdirectory and add whatever toolchains your project needs.

Each **subdirectory** is a purpose-built variant with its own readme covering build, run, and domain-specific notes.

| Variant | Directory | Description |
|---------|-----------|-------------|
| Template | `.` | Base images, no language toolchains |
| [Terraform](terraform/readme.md) | `terraform/` | Go + Terraform for provider development |

## Templates

### Claude

Build:

```bash
docker build -f Dockerfile.claude.template -t claude-code:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.claude.template -t claude-code:latest .
```

#### Authentication

**Option A: Databricks-routed credentials (`settings.json`)**

`~/.claude/settings.json` contains the env block (ANTHROPIC_BASE_URL, ANTHROPIC_AUTH_TOKEN, etc.) that Claude Code reads at startup. Mounted read-only so the container cannot modify it.

**Option B: Direct API key**

`ANTHROPIC_API_KEY` must be set in your shell (e.g. via `export ANTHROPIC_API_KEY=sk-...`). The `-e` flag passes it through without embedding it in the command.

#### Run modes

**Headless: run with a prompt**

Option A:
```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  claude-code:latest --prompt --verbose --dangerously-skip-permissions \
  "your prompt here"
```

Option B:
```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code:latest --prompt --verbose --dangerously-skip-permissions \
  "your prompt here"
```

**Interactive: launch Claude in the container**

Option A:
```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  claude-code:latest --verbose --dangerously-skip-permissions
```

Option B:
```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code:latest --verbose --dangerously-skip-permissions
```

To shorten the interactive command to a single word, add an alias to `~/.zshrc`:

```bash
alias danger='docker run -it --rm -v $(pwd):/workspace -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro claude-code:latest --verbose --dangerously-skip-permissions'
```

Then reload with `source ~/.zshrc` and run `danger` from any repo root.

### Gemini

Build:

```bash
docker build -f Dockerfile.gemini.template -t gemini-cli:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.gemini.template -t gemini-cli:latest .
```

#### Authentication

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

Build:

```bash
docker build -f Dockerfile.codex.template -t codex-cli:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.codex.template -t codex-cli:latest .
```

#### Authentication

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
