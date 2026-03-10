# Terraform Dev Sandboxes

These images extend the base coding agent sandboxes with Go and Terraform, tailored for Terraform provider development. The concept is the same — run the CLI inside a Docker container so the agent is isolated from your host system — but the image comes pre-loaded with the toolchain you need.

## 1. Build the image

### Claude

```bash
docker build -f Dockerfile.claude -t claude-code-tf:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.claude -t claude-code-tf:latest .
```

### Gemini

```bash
docker build -f Dockerfile.gemini -t gemini-cli-tf:latest .
```

On Apple Silicon:

```bash
docker build --platform linux/arm64 -f Dockerfile.gemini -t gemini-cli-tf:latest .
```

## 2. Launch the agent

Authentication and run modes are the same as the base templates (see [parent readme](../readme.md)). Substitute the appropriate image tag (`claude-code-tf:latest` or `gemini-cli-tf:latest`).

### Claude examples

**Headless (settings.json auth):**

```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  claude-code-tf:latest --prompt --verbose --dangerously-skip-permissions \
  "your prompt here"
```

**Interactive (API key auth):**

```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code-tf:latest --verbose --dangerously-skip-permissions
```

### Gemini examples

**Headless (credentials file):**

```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.gemini/.env:/home/agent/.gemini/.env:ro \
  gemini-cli-tf:latest --yolo \
  "your prompt here"
```

**Headless (API key):**

```bash
docker run -it --rm \
  -e GEMINI_API_KEY \
  -v $(pwd):/workspace \
  gemini-cli-tf:latest --yolo \
  "your prompt here"
```

## 3. Create a shell shortcut

Rather than typing the full `docker run` command each time, wrap it in a shell function. Add the following to your `~/.zshrc` (or `~/.bashrc`):

```bash
claudetf() {
  docker run -it --rm \
    --workdir $(pwd) \
    -v $(pwd):$(pwd) \
    -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
    -v /Users/jerome.lieow/Documents/GitHub/terraform-provider-databricks/.git:/Users/jerome.lieow/Documents/GitHub/terraform-provider-databricks/.git \
    -v /Users/jerome.lieow/Documents/GitHub/sandbox_workbricks/terraform/databricks_sandbox/local_provider:/local_provider \
    -v ~/.terraformrc:/home/agent/.terraformrc \
    claude-code-tf:latest --verbose --dangerously-skip-permissions "$@"
}
```

Key flags:

- `--workdir $(pwd)` and `-v $(pwd):$(pwd)` mount and set the working directory to the same absolute path as the host, so the agent sees identical paths.
- The `.git` volume mount enables git operations inside worktree checkouts (see below).
- The `local_provider` and `.terraformrc` mounts let Terraform use a locally-built provider binary instead of downloading from the registry.

Reload your shell and run from any worktree directory:

```bash
source ~/.zshrc

cd ~/Documents/GitHub/terraform-provider-databricks-fix-ignore-changes-computed
claudetf
```

Pass extra arguments as needed, e.g. `claudetf --prompt "your prompt here"`.

## Git worktrees

If the workspace is a git worktree, git operations inside the container will fail. A worktree's `.git` is a file pointing to the main repo's `.git/worktrees/<name>` directory, and that path doesn't exist inside the container.

To fix this, mount the main repo's `.git` directory at the same absolute path so the reference resolves:

```bash
docker run -it --rm \
  -v $(pwd):/workspace \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  -v /path/to/main-repo/.git:/path/to/main-repo/.git:ro \
  claude-code-tf:latest --verbose --dangerously-skip-permissions
```

Replace `/path/to/main-repo/.git` with the actual path. The host and container paths must match exactly so the gitdir reference resolves.

Working example for a `terraform-provider-databricks` worktree:

```bash
cd ~/Documents/GitHub/terraform-provider-databricks-fix-bug

docker run -it --rm \
  --workdir $(pwd) \
  -v $(pwd):$(pwd) \
  -v ~/.claude/settings.json:/home/agent/.claude/settings.json:ro \
  -v $HOME/Documents/GitHub/terraform-provider-databricks/.git:$HOME/Documents/GitHub/terraform-provider-databricks/.git:ro \
  -v ~/.terraformrc:/home/agent/.terraformrc \
  claude-code-tf:latest --verbose --dangerously-skip-permissions
```
