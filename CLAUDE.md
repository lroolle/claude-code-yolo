
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## We're following Issue-Based Development (IBD) workflow
1. Before running any Git/GitHub CLI `Bash` command (`git commit`, `gh issue create`, `gh pr create`, etc.), open the corresponding file in @workflows to review required steps.
2. Always apply the exact templates or conventions from the following files:
   - @workflows/GITHUB-ISSUE.md → issues
   - @workflows/GIT-COMMIT.md  → commits
   - @workflows/GITHUB-PR.md   → pull requests
3. Keep one branch per issue; merging the PR must auto-close its linked issue.


## PROJECT STRUCTURE

```
claude-code-yolo/
├── CLAUDE.md                  # This file - project guidance for Claude
├── README.md                  # Main documentation
├── CHANGELOG.md               # Version history
├── DEV-LOGS.md               # Development journal
├── TODO.md                   # Task tracking
├── install.sh                # One-line installer script
├── Makefile                  # Build automation
├── Dockerfile                # Container image definition
├── .dockerignore             # Docker build exclusions
├── docker-entrypoint.sh      # Container startup script
├── .gitignore                # Git exclusions
├── claude.sh                 # Main wrapper script (local/Docker modes)
├── claude-yolo               # Quick YOLO mode wrapper
├── claudeb.sh                # Bedrock authentication helper
│
├── .github/                  # GitHub automation
│   ├── ISSUE_TEMPLATE.md     # Bug report template
│   └── workflows/            # GitHub Actions
│       ├── ci.yml           # Pull request checks
│       ├── release.yml      # Release automation
│       ├── claude.yml       # Claude @mention handler
│       └── claude-code-review.yml  # Automated code review
│
├── workflows/                # Development workflows
│   ├── GIT-COMMIT.md        # Commit guidelines
│   ├── GITHUB-ISSUE.md      # Issue creation guidelines
│   ├── GITHUB-PR.md         # PR creation process
│
└── references/              # Documentation and research
    ├── chats/               # AI conversation logs
    ├── claude-code/         # Official Claude Code repo reference
    ├── claude-code-docs/    # Official documentation
    └── code-reviews/        # AI code review examples
```

## PROJECT PURPOSE

**Claude Code YOLO wraps the Claude CLI in Docker to safely enable `--dangerously-skip-permissions` without compromising your local machine.**

### The Problem

Claude Code CLI is excellent for code editing and development but has friction points:

1. **Cost Management**: Claude Pro/Max ($20-200/month) has rate limits. When exhausted, we switch to expensive API usage.
2. **Permission Prompts**: Claude's security design constantly asks for permission to edit files or access web. Annoying in trusted workspaces.
3. **Root Restriction**: Claude CLI refuses `--dangerously-skip-permissions` when running as root/sudo for security.

### Why YOLO Mode

We created `claude.sh --yolo` to solve the permission friction:
- Run Claude with `--dangerously-skip-permissions` for full workspace access
- Container provides the safety boundary instead of permission prompts
- No firewall restrictions, full web access within container

### Technical Challenges Solved

1. **Auth Persistence**: Mount `~/.claude` to avoid re-authentication. Claude hardcodes this path with no config option.
2. **Root Bypass**: Create non-root `claude` user to satisfy CLI security while maintaining full container capabilities.
3. **Path Consistency**: Mount identical directory structure so Claude remembers project contexts.
4. **Multi-Auth Support**: Seamless switching between Claude Pro, API keys, AWS Bedrock, and Vertex AI.

**Result**: Full Claude Code power with container isolation. No local machine risk, no permission fatigue.

## Project Architecture

Claude Code YOLO is a Docker-based wrapper for Claude Code CLI that provides safe execution with full development capabilities. The project implements a dual-mode architecture:

- **Local Mode** (default): Direct execution for speed
- **YOLO Mode** (`--yolo`): Docker containerized execution with `--dangerously-skip-permissions` safely enabled

### Core Components

- **`claude.sh`**: Main wrapper script that handles argument parsing, authentication selection, and mode switching
- **`claude-yolo`**: Quick wrapper script that runs `claude.sh --yolo` for convenient YOLO mode access
- **`Dockerfile`**: Ubuntu 24.04-based image with full development stack (Python 3.12, Node.js 22, Go 1.22, Rust, AWS CLI, dev tools)
- **`docker-entrypoint.sh`**: Container initialization script with environment reporting
- **Helper Scripts**: `claudeb.sh` (Bedrock authentication and model alias conversion)

### Authentication System

The wrapper supports 4 authentication methods with explicit control:
1. Claude App OAuth (`--claude`, `-c`) - Default, uses `~/.claude`
2. Anthropic API Key (`--api-key`, `-a`) - Direct API with model alias conversion
3. AWS Bedrock (`--bedrock`, `-b`) - Uses AWS credentials with model ARN mapping
4. Google Vertex AI (`--vertex`, `-v`) - Uses gcloud credentials

Model aliases are automatically converted to appropriate formats (API model names vs Bedrock ARNs) based on authentication method.

### Safety Features

**Dangerous Directory Detection**: The script prevents accidental execution in sensitive directories:
- Detects when running in home directory, system directories (`/`, `/etc`, `/usr`), or other critical paths
- Shows a clear warning about Claude's full access to the directory
- Requires explicit confirmation (`yes`) to proceed
- Protects users from accidentally giving Claude access to all personal files

## Docker Architecture Details

### Volume Mounting Strategy
- Current directory mounted at identical path (preserves absolute paths)
- Authentication directories mounted to `/root/`: `~/.claude`, `~/.aws`, `~/.config/gcloud`
- Project-specific `.claude` settings mounted when present
- Docker socket mounted for Docker-in-Docker scenarios

### Container Configuration
- Runs as non-root user matching your host UID/GID (for file access)
- Uses `tini` as PID 1 for proper signal handling
- Automatic proxy translation: `localhost` → `host.docker.internal`
- Environment variable pass-through for development tools
- **Automatically adds `--dangerously-skip-permissions` in YOLO mode for full power**
- Copies and sets proper permissions on authentication files (`~/.claude`, `~/.claude.json`)

## Key Implementation Patterns

### Authentication File Handling
Claude requires both `~/.claude/` directory AND `~/.claude.json` file - both must be mounted for OAuth to work properly.

### Proxy Translation
Docker container automatically translates `127.0.0.1` and `localhost` to `host.docker.internal` in proxy URLs for container-to-host communication.

### Model Alias System
- Bedrock: Converts aliases to full ARNs (`sonnet-4` → `arn:aws:bedrock:us-west-2:123456789012:inference-profile/us.anthropic.claude-sonnet-4-20250514-v1:0`)
- API: Converts aliases to model names (`sonnet-4` → `claude-sonnet-4-20250514`)
- Supports all current Claude models plus DeepSeek R1

### Error Handling
- Missing executables trigger fallback checks and helpful error messages
- Authentication issues can be debugged by running with `--trace` flag
- Docker image auto-builds when missing

## Common Issues Solutions

* "Invalid API key" - Mount both `~/.claude` and `~/.claude.json`
* Proxy not working - Localhost auto-translates to `host.docker.internal`
* Permission denied - Container copies auth files with proper permissions
* Claude refuses dangerous permissions on root user - YOLO mode runs as non-root + adds `--dangerously-skip-permissions`
* Running in home directory - Script warns and requires confirmation, always cd to project first

## Development Tools Included

**Languages**: Python 3.12, Node.js 22 (via NodeSource), Go 1.22, Rust
**Build Tools**: make, cmake, gcc, g++, clang
**CLI Tools**: git, docker, aws, jq, ripgrep, fzf, bat, eza, delta
**Editors**: vim, neovim, nano
**Shell**: zsh with oh-my-zsh and plugins
**Claude**: claude, claude-trace pre-installed
