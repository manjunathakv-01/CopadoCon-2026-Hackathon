# CopadoCon-2026-Hackathon
Dream - Think - Build


# Phase 1: Project Initialization & Go Daemon Core

Let's build this ground-up, one file at a time. I'll explain every decision.

  ![image alt](copado_phantom_architecture.svg)


---

## Step 1: Folder Structure

Create this directory tree manually first. Nothing gets written to it yet — just `mkdir` everything so Go's tooling is happy from the start.

```
copado-phantom/
├── README.md
├── SKILL.md                          ← (Phase 4, placeholder for now)
├── Makefile
├── go.mod
├── go.sum
├── phantom.toml.example
│
├── cmd/
│   ├── phantom/
│   │   └── main.go                   ← Daemon entrypoint
│   └── phantom-cli/
│       └── main.go                   ← CLI called by Git hooks
│
├── internal/
│   ├── config/
│   │   └── config.go                 ← TOML config loader
│   ├── state/
│   │   ├── cache.go                  ← BoltDB wrapper
│   │   └── schema.go                 ← Bucket names + typed keys
│   ├── ipc/
│   │   ├── server.go                 ← Unix socket server (daemon)
│   │   └── client.go                 ← Unix socket client (CLI)
│   ├── copado/                       ← Phase 3
│   ├── dialogue/                     ← Phase 3
│   ├── crt/                          ← Phase 3
│   ├── hooks/                        ← Phase 2
│   └── tui/                          ← Phase 5
│
├── mcp-server/                       ← Phase 4 (TypeScript/Bun)
│   ├── index.ts
│   ├── package.json
│   └── tools/
│
└── scripts/
    ├── install-hooks.sh
    └── uninstall.sh
```

Run this to create it all at once:

```bash
mkdir -p copado-phantom/{cmd/{phantom,phantom-cli},internal/{config,state,ipc,copado,dialogue,crt,hooks,tui},mcp-server/tools,scripts}
cd copado-phantom
```

---

