# Claude Multiplay

A Claude Code skill for seamless pair programming with automatic git handoffs and live dev server sharing.

## Features

- **Automatic Git Handoffs**: Use `handoff` and `takeover` triggers for one-command turn-taking
- **Live Dev Server Sharing**: Automatically creates Cloudflare tunnels when dev servers start
- **Smart Sync**: Auto-detects when behind remote and syncs before pushing
- **Zero Config Tunnels**: No account or auth token required for sharing

## Installation

### Option 1: Install via Claude Code CLI

```bash
# Install the skill
claude-code skill add https://github.com/kaalund/claude-multiplay

# Verify installation
claude-code skill list
```

### Option 2: Manual Installation

1. Create the skills directory:
```bash
mkdir -p ~/.claude/skills/multiplay
```

2. Download the skill:
```bash
curl -o ~/.claude/skills/multiplay/SKILL.md \
  https://raw.githubusercontent.com/kaalund/claude-multiplay/main/multiplay.md
```

3. Restart Claude Code

## Quick Start

### 1. Set Up Shared Repository

Both developers need access to the same git repository:

```bash
# Create a repo on GitHub/GitLab
# Then connect your local repo:
git remote add origin YOUR_REPO_URL
git push -u origin main
```

### 2. Start Pair Programming

**Developer A (Anders):**
```bash
# Start your dev server
npm run dev

# Claude automatically detects and creates shareable link
# Share the link with your partner

# When done coding:
# Say: "handoff"
```

**Developer B (Bob):**
```bash
# Open the link Anders shared to see live changes

# When Anders hands off:
# Say: "takeover"

# Start your own dev server
npm run dev

# Claude creates a new shareable link for Anders
```

## Commands

| Trigger | What it does |
|---------|-------------|
| `handoff` | Stage, commit, push your changes |
| `takeover` | Pull latest changes and start coding |
| `share dev server` | Manually create tunnel (usually auto-detected) |

## How It Works

1. **Anders codes** → Runs dev server (Vite, Next.js, etc.)
2. **Claude detects server** → Auto-creates Cloudflare tunnel
3. **Bob watches live** → Opens shared URL, sees changes with hot reload
4. **Anders finishes** → Says "handoff" (commits & pushes)
5. **Bob takes over** → Says "takeover" (pulls code)

## Requirements

- Git repository with remote configured
- Homebrew (for cloudflared installation)
- Claude Code CLI or Desktop App

## Troubleshooting

### No remote configured
```bash
# Check remote:
git remote -v

# If empty, add remote:
git remote add origin YOUR_REPO_URL
```

### Cloudflared not installed
Claude will auto-install, or install manually:
```bash
brew install cloudflare/cloudflare/cloudflared
```

### Port detection failed
Manually specify:
```bash
cloudflared tunnel --url http://localhost:YOUR_PORT
```

## Contributing

Contributions welcome! Please open an issue or PR.

## License

MIT - See [LICENSE](LICENSE) file for details
