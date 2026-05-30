# CopadoCon-2026-Hackathon
Dream - Think - Build


# Phase 1: Project Initialization & Go Daemon Core

Let's build this ground-up, one file at a time. I'll explain every decision.

<img src="https://github.com/user-attachments/assets/54cc8b95-6fff-461d-b4e8-7fc20a040c51" alt="copado_phantom_architecture" width="124" height="150" />



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

## Step 2: README.md

**File:** `copado-phantom/README.md`

```markdown
# Copado Phantom

> Autonomous Headless Shadow-Branching DevOps Daemon for Salesforce teams.

Copado Phantom runs as a local background daemon that intercepts native Git
commands (`git checkout`, `git commit`, `git push`) and transparently
orchestrates the full Copado DevOps lifecycle — without opening a browser tab.

## What It Does

| Git Command         | Phantom Action                                              |
|---------------------|-------------------------------------------------------------|
| `git checkout -b`   | Syncs User Story metadata via Copado REST API               |
| `git commit`        | Pauses commit, sends diff to Build Agent for security review|
| `git push`          | Triggers UAT validation + launches live TUI dashboard       |

## Architecture

```
Local Repo → Git Hooks → phantom-cli → IPC Socket → Phantom Daemon
                                                          │
                    ┌─────────────────────────────────────┤
                    │                                     │
             Copado REST API                    Dialogue API Agents
             CRT Open API                       (Plan/Build/Test/Release)
                    │                                     │
                    └──────────── BoltDB Cache ───────────┘
                                       │
                              MCP Server (Bun/TS)
                          Claude Code / Cursor integration
```

## Tech Stack

| Layer            | Technology                                      |
|------------------|-------------------------------------------------|
| Core Daemon      | Go 1.22+, BoltDB, Bubble Tea, Resty             |
| Git Hooks        | Bash + Go subprocess (`phantom-cli`)            |
| MCP Server       | TypeScript, Bun, `@modelcontextprotocol/sdk`    |
| IPC              | Unix domain socket (JSON-RPC over socket)       |
| Config           | TOML (`phantom.toml`)                           |

## Quick Start

```bash
# 1. Build the daemon and CLI
make build

# 2. Start the daemon (runs in background)
phantom start

# 3. Install hooks into a target Git repository
phantom-cli install-hooks --repo /path/to/your/sf-project

# 4. Verify everything is wired correctly
phantom doctor

# 5. Start the MCP server (for Claude Code / Cursor)
cd mcp-server && bun run index.ts
```

## Configuration

Copy `phantom.toml.example` to `phantom.toml` and fill in your credentials:

```toml
[copado]
org_url    = "https://your-org.copado.com"
api_token  = "YOUR_COPADO_API_TOKEN"
pipeline_id = "YOUR_PIPELINE_ID"

[crt]
token      = "YOUR_CRT_TOKEN"
project_id = "YOUR_CRT_PROJECT_ID"

[daemon]
socket_path = "/tmp/copado-phantom.sock"
log_level   = "info"
```

## Hackathon Track

- **Primary:** Track C — Your Headless Idea
- **Bonus:**   Track B — SKILL.md & MCP Server

Submission deadline: **3rd June 2026** | Demo Jam: **13th June 2026, Bengaluru**
```

---

## Step 3: go.mod

**File:** `copado-phantom/go.mod`

```go
module github.com/yourorg/copado-phantom

go 1.22

require (
    github.com/BurntSushi/toml v1.3.2
    github.com/charmbracelet/bubbletea v0.26.4
    github.com/charmbracelet/lipgloss v0.11.0
    github.com/go-resty/resty/v2 v2.13.1
    go.etcd.io/bbolt v1.3.9
)
```

Run after creating this file:

```bash
cd copado-phantom
go mod tidy
```

---

## Step 4: phantom.toml.example

**File:** `copado-phantom/phantom.toml.example`

```toml
# Copado Phantom Configuration
# Copy this file to phantom.toml and fill in your values.
# phantom.toml is .gitignored — never commit real tokens.

[copado]
org_url     = "https://your-org.copado.com"
api_token   = "YOUR_COPADO_BEARER_TOKEN"
pipeline_id = "a1B000000000001AAA"

[crt]
token      = "YOUR_CRT_API_TOKEN"
project_id = "YOUR_CRT_PROJECT_UUID"

[daemon]
socket_path = "/tmp/copado-phantom.sock"
log_level   = "info"          # debug | info | warn | error
db_path     = "/tmp/copado-phantom.db"
```

---

## Step 5: Config Loader

**File:** `copado-phantom/internal/config/config.go`

This is the single source of truth for runtime configuration. Every other package imports this — nothing reads `phantom.toml` directly.

```go
package config

import (
	"fmt"
	"os"

	"github.com/BurntSushi/toml"
)

// Config is the root configuration structure.
// Field names map 1:1 to phantom.toml sections.
type Config struct {
	Copado CopadoConfig `toml:"copado"`
	CRT    CRTConfig    `toml:"crt"`
	Daemon DaemonConfig `toml:"daemon"`
}

type CopadoConfig struct {
	OrgURL     string `toml:"org_url"`
	APIToken   string `toml:"api_token"`
	PipelineID string `toml:"pipeline_id"`
}

type CRTConfig struct {
	Token     string `toml:"token"`
	ProjectID string `toml:"project_id"`
}

type DaemonConfig struct {
	SocketPath string `toml:"socket_path"`
	LogLevel   string `toml:"log_level"`
	DBPath     string `toml:"db_path"`
}

// Load reads phantom.toml from the given path.
// Environment variables override TOML values — useful for CI/CD contexts.
func Load(path string) (*Config, error) {
	cfg := &Config{
		// Sensible defaults — overridden by TOML or env vars below.
		Daemon: DaemonConfig{
			SocketPath: "/tmp/copado-phantom.sock",
			LogLevel:   "info",
			DBPath:     "/tmp/copado-phantom.db",
		},
	}

	if _, err := os.Stat(path); os.IsNotExist(err) {
		return nil, fmt.Errorf("config file not found at %q — copy phantom.toml.example to phantom.toml", path)
	}

	if _, err := toml.DecodeFile(path, cfg); err != nil {
		return nil, fmt.Errorf("failed to parse config file %q: %w", path, err)
	}

	// Environment variable overrides (useful in CI pipelines)
	if v := os.Getenv("PHANTOM_COPADO_TOKEN"); v != "" {
		cfg.Copado.APIToken = v
	}
	if v := os.Getenv("PHANTOM_CRT_TOKEN"); v != "" {
		cfg.CRT.Token = v
	}
	if v := os.Getenv("PHANTOM_SOCKET"); v != "" {
		cfg.Daemon.SocketPath = v
	}

	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}

// validate enforces required fields early — fail loudly at startup,
// not mid-operation when a developer is waiting on a git commit.
func (c *Config) validate() error {
	if c.Copado.OrgURL == "" {
		return fmt.Errorf("config: copado.org_url is required")
	}
	if c.Copado.APIToken == "" {
		return fmt.Errorf("config: copado.api_token is required (or set PHANTOM_COPADO_TOKEN env var)")
	}
	if c.Copado.PipelineID == "" {
		return fmt.Errorf("config: copado.pipeline_id is required")
	}
	return nil
}
```

---

## Step 6: BoltDB Schema

**File:** `copado-phantom/internal/state/schema.go`

BoltDB is a key-value store organized into "buckets" — think of each bucket as a table. We define all bucket names and key patterns here so they never get scattered across the codebase.

```go
package state

// Bucket names — one per logical domain.
// BoltDB creates these automatically on first write.
const (
	// BucketUserStories stores Copado User Story JSON keyed by US ID.
	// Key: "US-1234"  →  Value: JSON(UserStory)
	BucketUserStories = "user_stories"

	// BucketBranchMap maps local Git branch names to Copado US IDs.
	// Key: "feature/US-1234-add-apex-trigger"  →  Value: "US-1234"
	BucketBranchMap = "branch_map"

	// BucketJobIDs tracks active Copado job execution IDs per branch.
	// Key: "feature/US-1234:validate"  →  Value: "job-execution-id-xyz"
	BucketJobIDs = "job_ids"

	// BucketPlanVerdicts caches the Plan agent conflict analysis per branch.
	// Key: "feature/US-1234"  →  Value: JSON(PlanVerdict)
	BucketPlanVerdicts = "plan_verdicts"

	// BucketMeta stores singleton daemon metadata (last sync time, version).
	// Key: "daemon_version"  →  Value: "1.0.0"
	BucketMeta = "meta"

	// BucketErrors is an append-log of non-fatal daemon errors.
	// Key: RFC3339 timestamp  →  Value: error message string
	BucketErrors = "errors"
)

// JobType qualifies what kind of job ID is stored in BucketJobIDs.
type JobType string

const (
	JobTypeValidate JobType = "validate"
	JobTypeCommit   JobType = "commit"
	JobTypeCRT      JobType = "crt"
)

// JobKey builds the composite key for BucketJobIDs.
// e.g. "feature/US-1234:validate"
func JobKey(branch string, jt JobType) string {
	return branch + ":" + string(jt)
}
```

---

## Step 7: BoltDB Cache Wrapper

**File:** `copado-phantom/internal/state/cache.go`

This is the core storage engine. Every other package talks to BoltDB exclusively through this interface — no other package imports bbolt directly.

```go
package state

import (
	"encoding/json"
	"fmt"
	"time"

	bolt "go.etcd.io/bbolt"
)

// Cache wraps a BoltDB database with typed read/write methods.
// All methods are safe for concurrent use — BoltDB serializes writes internally.
type Cache struct {
	db *bolt.DB
}

// Open opens (or creates) the BoltDB file at the given path
// and ensures all required buckets exist.
func Open(path string) (*Cache, error) {
	db, err := bolt.Open(path, 0600, &bolt.Options{Timeout: 2 * time.Second})
	if err != nil {
		return nil, fmt.Errorf("state: could not open db at %q: %w", path, err)
	}

	// Create all buckets in a single setup transaction.
	// bolt.Update is a read-write transaction; it commits on nil return.
	err = db.Update(func(tx *bolt.Tx) error {
		buckets := []string{
			BucketUserStories,
			BucketBranchMap,
			BucketJobIDs,
			BucketPlanVerdicts,
			BucketMeta,
			BucketErrors,
		}
		for _, name := range buckets {
			if _, err := tx.CreateBucketIfNotExists([]byte(name)); err != nil {
				return fmt.Errorf("create bucket %q: %w", name, err)
			}
		}
		return nil
	})
	if err != nil {
		return nil, fmt.Errorf("state: bucket initialization failed: %w", err)
	}

	return &Cache{db: db}, nil
}

// Close flushes pending writes and closes the database file.
// Always defer this after Open.
func (c *Cache) Close() error {
	return c.db.Close()
}

// ── User Story ────────────────────────────────────────────────────────────────

// UserStory is the minimal Copado US shape we cache locally.
// The full API response gets stored as JSON — we decode only what we need.
type UserStory struct {
	ID           string   `json:"id"`
	Name         string   `json:"name"`
	Status       string   `json:"status"`
	PipelineName string   `json:"pipeline_name"`
	PipelineURL  string   `json:"pipeline_url"`
	TargetBranch string   `json:"target_branch"`
	MetadataScope []string `json:"metadata_scope"`
	Sprint       string   `json:"sprint"`
	SyncedAt     time.Time `json:"synced_at"`
}

// PutUserStory serializes and stores a UserStory.
func (c *Cache) PutUserStory(us *UserStory) error {
	return c.putJSON(BucketUserStories, us.ID, us)
}

// GetUserStory retrieves a UserStory by its Copado ID (e.g. "US-1234").
// Returns nil, nil if not found — callers must check for nil.
func (c *Cache) GetUserStory(id string) (*UserStory, error) {
	var us UserStory
	found, err := c.getJSON(BucketUserStories, id, &us)
	if err != nil || !found {
		return nil, err
	}
	return &us, nil
}

// ── Branch → US Mapping ───────────────────────────────────────────────────────

// PutBranchMapping records which US ID a local branch belongs to.
func (c *Cache) PutBranchMapping(branch, usID string) error {
	return c.putString(BucketBranchMap, branch, usID)
}

// GetUSIDForBranch returns the Copado US ID mapped to a branch.
// Returns "", nil if no mapping exists.
func (c *Cache) GetUSIDForBranch(branch string) (string, error) {
	return c.getString(BucketBranchMap, branch)
}

// GetUserStoryForBranch is a convenience method combining the two lookups above.
func (c *Cache) GetUserStoryForBranch(branch string) (*UserStory, error) {
	usID, err := c.GetUSIDForBranch(branch)
	if err != nil || usID == "" {
		return nil, err
	}
	return c.GetUserStory(usID)
}

// ── Job IDs ───────────────────────────────────────────────────────────────────

// PutJobID stores a Copado job execution ID for a branch+type combo.
func (c *Cache) PutJobID(branch string, jt JobType, jobID string) error {
	return c.putString(BucketJobIDs, JobKey(branch, jt), jobID)
}

// GetJobID retrieves the stored job execution ID.
func (c *Cache) GetJobID(branch string, jt JobType) (string, error) {
	return c.getString(BucketJobIDs, JobKey(branch, jt))
}

// ── Plan Verdicts ─────────────────────────────────────────────────────────────

// PlanVerdict is the Plan agent's conflict analysis result.
type PlanVerdict struct {
	HasBlockers    bool      `json:"hasBlockers"`
	Blockers       []string  `json:"blockers"`
	Warnings       []string  `json:"warnings"`
	Recommendation string    `json:"recommendation"`
	EvaluatedAt    time.Time `json:"evaluatedAt"`
}

// PutPlanVerdict caches a Plan agent response for a branch.
func (c *Cache) PutPlanVerdict(branch string, v *PlanVerdict) error {
	return c.putJSON(BucketPlanVerdicts, branch, v)
}

// GetPlanVerdict retrieves the cached plan verdict for a branch.
func (c *Cache) GetPlanVerdict(branch string) (*PlanVerdict, error) {
	var v PlanVerdict
	found, err := c.getJSON(BucketPlanVerdicts, branch, &v)
	if err != nil || !found {
		return nil, err
	}
	return &v, nil
}

// ── Error Log ─────────────────────────────────────────────────────────────────

// AppendError logs a non-fatal daemon error with a timestamp key.
func (c *Cache) AppendError(msg string) error {
	key := time.Now().UTC().Format(time.RFC3339Nano)
	return c.putString(BucketErrors, key, msg)
}

// ── Low-level helpers ─────────────────────────────────────────────────────────

func (c *Cache) putJSON(bucket, key string, v any) error {
	data, err := json.Marshal(v)
	if err != nil {
		return fmt.Errorf("state: marshal %q: %w", key, err)
	}
	return c.db.Update(func(tx *bolt.Tx) error {
		return tx.Bucket([]byte(bucket)).Put([]byte(key), data)
	})
}

func (c *Cache) getJSON(bucket, key string, dest any) (bool, error) {
	var data []byte
	err := c.db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket([]byte(bucket)).Get([]byte(key))
		if v == nil {
			return nil // key not found — not an error
		}
		data = make([]byte, len(v))
		copy(data, v) // must copy: v is only valid inside the transaction
		return nil
	})
	if err != nil || data == nil {
		return false, err
	}
	return true, json.Unmarshal(data, dest)
}

func (c *Cache) putString(bucket, key, value string) error {
	return c.db.Update(func(tx *bolt.Tx) error {
		return tx.Bucket([]byte(bucket)).Put([]byte(key), []byte(value))
	})
}

func (c *Cache) getString(bucket, key string) (string, error) {
	var result string
	err := c.db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket([]byte(bucket)).Get([]byte(key))
		if v != nil {
			result = string(v)
		}
		return nil
	})
	return result, err
}
```

---

## Step 8: Daemon Entrypoint

**File:** `copado-phantom/cmd/phantom/main.go`

This is the process that runs permanently in the background. It opens the DB, starts the IPC socket server, and blocks until SIGTERM.

```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/yourorg/copado-phantom/internal/config"
	"github.com/yourorg/copado-phantom/internal/state"
)

func main() {
	// ── 1. Locate config file ─────────────────────────────────────────────────
	cfgPath := os.Getenv("PHANTOM_CONFIG")
	if cfgPath == "" {
		cfgPath = "phantom.toml"
	}

	cfg, err := config.Load(cfgPath)
	if err != nil {
		log.Fatalf("phantom: config error: %v", err)
	}

	// ── 2. Open BoltDB ────────────────────────────────────────────────────────
	cache, err := state.Open(cfg.Daemon.DBPath)
	if err != nil {
		log.Fatalf("phantom: database error: %v", err)
	}
	defer func() {
		if err := cache.Close(); err != nil {
			log.Printf("phantom: warning — db close error: %v", err)
		}
	}()

	log.Printf("phantom: daemon started (db=%s, socket=%s)",
		cfg.Daemon.DBPath, cfg.Daemon.SocketPath)

	// ── 3. Handle OS signals for graceful shutdown ────────────────────────────
	// SIGTERM is sent by `kill` or systemd stop.
	// SIGINT is sent by Ctrl+C during development.
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

	// ── 4. Block until shutdown signal ───────────────────────────────────────
	// Phase 1: just signal handling.
	// Phase 2: IPC socket server starts here.
	// Phase 5: TUI launches here.
	sig := <-sigCh
	fmt.Printf("\nphantom: received %s — shutting down cleanly\n", sig)
}
```

---

## Step 9: Makefile

**File:** `copado-phantom/Makefile`

```makefile
BINARY_DAEMON  := phantom
BINARY_CLI     := phantom-cli
BUILD_DIR      := ./build

.PHONY: build clean run doctor

build:
	@echo "→ Building daemon..."
	go build -o $(BUILD_DIR)/$(BINARY_DAEMON) ./cmd/phantom
	@echo "→ Building CLI..."
	go build -o $(BUILD_DIR)/$(BINARY_CLI) ./cmd/phantom-cli
	@echo "✔ Build complete: $(BUILD_DIR)/"

run:
	@echo "→ Starting Phantom daemon..."
	PHANTOM_CONFIG=phantom.toml $(BUILD_DIR)/$(BINARY_DAEMON)

clean:
	rm -rf $(BUILD_DIR)
	rm -f /tmp/copado-phantom.sock /tmp/copado-phantom.db

doctor:
	@echo "→ Checking Phantom installation..."
	@test -f phantom.toml && echo "  ✔ phantom.toml found" || echo "  ✘ phantom.toml missing"
	@test -S /tmp/copado-phantom.sock && echo "  ✔ daemon socket active" || echo "  ✘ daemon not running"
	@go build ./... && echo "  ✔ all packages compile" || echo "  ✘ build errors found"
```

---

## Phase 1 Verification

Run these commands to confirm Phase 1 is clean:

```bash
# From copado-phantom/
go mod tidy
go build ./...       # must produce zero errors
make build           # produces build/phantom and build/phantom-cli
make run             # should print: "phantom: daemon started..."
# Ctrl+C → should print: "phantom: received interrupt — shutting down cleanly"
```

---

## What We Have So Far

| File | Purpose |
|---|---|
| `internal/config/config.go` | TOML loader with env var overrides and validation |
| `internal/state/schema.go` | All BoltDB bucket names and key patterns |
| `internal/state/cache.go` | Typed read/write API over BoltDB |
| `cmd/phantom/main.go` | Daemon entrypoint with graceful shutdown |
| `phantom.toml.example` | Config template |
| `Makefile` | Build, run, clean, doctor targets |

**Confirm Phase 1 builds cleanly, then reply and we move to Phase 2** — the Git hook interception engine, starting with the IPC socket so the daemon can receive hook events from `phantom-cli`.
