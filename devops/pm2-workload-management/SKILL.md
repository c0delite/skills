---
name: pm2-workload-management
description: "PM2 process management for Node.js workspaces: ecosystem configs, NVM compatibility, ESM/CJS, startup on reboot, and common failure modes."
version: 1.0.0
metadata:
  hermes:
    tags: [pm2, process-management, nodejs, nvm, background-services, devops]
    related_skills: [kanban-orchestrator, kanban-worker]
---

# PM2 Workload Management

## Overview

PM2 keeps Node.js processes running in the background, surviving terminal close and reboots. This skill covers the patterns that actually work — specifically around NVM-managed Node versions, ESM workspaces, and long-running dev servers.

## When to Use

- User wants a `pnpm dev` / `npm run dev` / `node server.js` process to survive terminal close
- Workspace needs auto-restart on crash
- Process needs to survive system reboot
- Project uses NVM-managed Node and PM2 runs with wrong version

## The PATH Problem (most common failure)

**Symptom:** PM2 starts a process but it crashes immediately with "You are using Node.js 18.19.1. Vite requires Node.js version 20.19+ or 22.12+."

**Root cause:** `#!/usr/bin/env node` in pnpm/npm shims resolves to the system Node (often 18.x), not the NVM-managed version. PM2 does NOT inherit the user's shell NVM setup.

**Fix:** Set `env.PATH` explicitly in ecosystem.config.cjs, putting the correct NVM bin dir FIRST:

```js
module.exports = {
  apps: [{
    name: 'my-workspace',
    script: '/home/user/.nvm/versions/node/v22.22.2/bin/pnpm',
    args: 'dev',
    cwd: '/home/user/workspace',
    interpreter: 'none',
    env: {
      NODE_OPTIONS: '--max-old-space-size=2048',
      PATH: '/home/user/.nvm/versions/node/v22.22.2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
    },
    watch: false,
    autorestart: true,
    max_restarts: 10,
    exp_backoff_restart_delay: 100
  }]
};
```

Key points:
- `script` points to the pnpm or node binary directly
- `interpreter: 'none'` — don't let PM2 try to interpret the script
- `env.PATH` must include the NVM bin dir at the START so `env node` resolves to the right version
- Use `.cjs` extension (not `.js`) when the project has `"type": "module"` in package.json

## ecosystem.config.cjs Template

For a typical Vite/pnpm workspace:

```js
module.exports = {
  apps: [{
    name: 'workspace-name',
    script: '/path/to/nvm/versions/node/vXX.x.x/bin/pnpm',
    args: 'dev',
    cwd: '/absolute/path/to/workspace',
    interpreter: 'none',
    env: {
      NODE_OPTIONS: '--max-old-space-size=2048',
      PATH: '/path/to/nvm/versions/node/vXX.x.x/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
    },
    watch: false,
    autorestart: true,
    max_restarts: 10,
    exp_backoff_restart_delay: 100
  }]
};
```

## ESM vs CJS for ecosystem config

| Project package.json | ecosystem config extension | module.exports |
|---|---|---|
| `"type": "module"` | `.config.cjs` | `module.exports = {...}` (CJS) |
| No `"type"` or `"commonjs"` | `.config.js` | `module.exports = {...}` (CJS) |
| `"type": "module"` | `.config.js` | FAILS — treated as ESM, require() unavailable |

**Rule:** When in doubt, use `.config.cjs`.

## Finding the right Node/pnpm path

```bash
# Find all node versions via NVM
ls ~/.nvm/versions/node/

# Get the pnpm path for a specific version
/home/user/.nvm/versions/node/v22.22.2/bin/pnpm

# Check what node a shim resolves to
cat /home/user/.nvm/versions/node/v22.22.2/bin/pnpm
# The shebang is #!/usr/bin/env node — THIS is what PATH fixes

# Verify Node version
/home/user/.nvm/versions/node/v22.22.2/bin/node --version
```

## Common PM2 Commands

| Command | Purpose |
|---|---|
| `pm2 start ecosystem.config.cjs` | Start process from config |
| `pm2 list` | Check status |
| `pm2 logs <name>` | View logs |
| `pm2 restart <name>` | Restart |
| `pm2 stop <name>` | Stop |
| `pm2 delete <name>` | Remove from PM2 |
| `pm2 save` | Save current process list |
| `pm2 startup` | Print systemd init command |
| `pm2 delete all` | Remove all processes |

## Reboot Survival

After `pm2 save`, run the systemd command it prints:

```bash
sudo env PATH=$PATH:/usr/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u sysadm --hp /home/sysadm
```

This creates a systemd service that starts PM2 and all saved processes on boot.

## Diagnostic Checklist

When PM2 keeps restarting:

1. **Check logs first:** `pm2 logs <name> --nostream --lines 50`
2. **Node version mismatch?** Look for "You are using Node.js X.X.X. Project requires Y.Y.Y+"
3. **PATH wrong?** `env node --version` in PM2 context should match `which node` in your shell
4. **ESM/CJS mismatch?** ecosystem.config.js with `"type": "module"` project needs `.cjs` extension
5. **Port already in use?** Vite/Express logs "Port 3000 is in use" — not fatal, it picks another
6. **Heap out of memory?** Add `NODE_OPTIONS: '--max-old-space-size=4096'` (in MB)

## PM2 vs nohup

| | PM2 | nohup |
|---|---|---|
| Auto-restart on crash | Yes | No |
| Log management | Yes (`pm2 logs`) | Manual redirect |
| Reboot survival | Yes (with startup) | No |
| Process list | Yes | No |
| Cluster mode | Yes | No |

Use PM2 for anything that needs to stay up. Use nohup for one-off long-running commands.
