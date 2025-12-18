# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

autobrr is a download automation tool for torrents and Usenet. It monitors IRC announce channels and RSS/Torznab/Newznab feeds in real-time, filters releases based on user-defined criteria, and automatically pushes them to download clients or \*arr applications for early swarm participation.

**Tech Stack:**
- Backend: Go 1.25+ with Chi router, zerolog
- Frontend: React 19 with TypeScript, Vite, TanStack Router/Query, Tailwind CSS
- Database: SQLite (embedded) or PostgreSQL
- Package Manager: pnpm (frontend)

## Development Commands

### Backend

```bash
# Install Go dependencies
go mod download

# Run backend (API server on http://localhost:7474)
go run cmd/autobrr/main.go

# Build backend binary to ./bin/autobrr
make build/app

# Build autobrrctl CLI tool
make build/ctl

# Run tests (excludes integration tests)
go test ./...

# Run all tests including integration tests (requires Docker)
docker compose up -d test_postgres
go test ./... -tags=integration

# Build Docker image (tagged as autobrr:dev)
make build/docker
```

### Frontend

```bash
# Install dependencies
cd web && pnpm install

# Run dev server (http://localhost:3000)
pnpm dev

# Build production frontend
pnpm run build

# Lint
pnpm lint
```

### Full Stack

```bash
# Build both frontend and backend
make build

# Run in development mode with tmux (requires tmux)
make dev
```

### Testing

```bash
# Run unit tests only (fast)
go test ./...

# Run integration tests (requires test_postgres container)
docker compose up -d test_postgres
go test ./... -tags=integration

# Run single package tests
go test ./internal/filter -v

# Run tests with timezone UTC (for integration test stability)
TZ=UTC go test ./... -tags=integration
```

## Architecture Overview

autobrr follows a **layered service-oriented architecture** with clear separation of concerns.

### Core Data Flow: IRC → Release → Filter → Action

1. **IRC Monitoring** (`internal/irc/`) - IRC handlers connect to tracker announce channels
2. **Announce Processing** (`internal/announce/`) - Parse announce messages using indexer-specific regex patterns
3. **Release Processing** (`internal/release/`) - Create Release objects with parsed metadata, normalize for duplicate detection
4. **Filter Matching** (`internal/filter/`) - Check releases against user-defined filters with regex/wildcard support
5. **Action Execution** (`internal/action/`) - Execute actions (push to clients, run scripts, webhooks) when filters match

### Backend Structure

The backend follows **Repository Pattern** and **Service Layer Pattern**:

```
internal/
├── domain/          # Core domain models and business logic (Release, Filter, Action, Indexer)
├── database/        # Repository implementations (SQLite & PostgreSQL)
├── <service>/       # Service layer (business logic orchestration)
│   ├── service.go   # Service implementation
│   └── ...
├── http/            # HTTP handlers and API routes
└── server/          # Main server orchestration
```

**Key Patterns:**
- **Repository Interfaces** - Defined in `internal/domain/`, implemented in `internal/database/`
- **Dependency Injection** - Constructor-based DI throughout (services receive dependencies as interfaces)
- **Event-Driven** - EventBus for loosely-coupled communication between services
- **Domain-Driven Design** - Rich domain models with business logic (e.g., `Filter.CheckFilter()`, `Release.ParseString()`)

### Main Components

**Domain Services** (in `internal/`):
- `action/` - Execute actions on matched releases (push to clients, exec, webhooks)
- `announce/` - Process IRC announce messages with indexer-specific parsers
- `download_client/` - Download client integrations (qBittorrent, Deluge, rTorrent, Transmission, Porla, SABnzbd)
- `feed/` - RSS/Torznab/Newznab feed polling and parsing
- `filter/` - Filtering engine with regex/wildcard matching, size checks, external filters
- `indexer/` - Indexer definitions and API integrations (75+ trackers)
- `irc/` - IRC client handlers for real-time monitoring with SASL/NickServ auth
- `list/` - List management for batch operations
- `notification/` - Notification service (Discord, Telegram, Notifiarr, etc.)
- `release/` - Release processing, parsing, normalization, duplicate detection
- `releasedownload/` - Torrent file download service

**Infrastructure Services**:
- `auth/` - Authentication (built-in + OpenID Connect)
- `config/` - Configuration management with dynamic reload
- `database/` - Repository implementations + migrations
- `http/` - API handlers and routes
- `logger/` - Structured logging with SSE integration for web UI
- `scheduler/` - Cron-based task scheduling
- `server/` - Main server startup and orchestration

### Application Initialization

From `cmd/autobrr/main.go`:

1. Config loading (env vars + config file)
2. Logger initialization with SSE stream
3. Database connection (SQLite or PostgreSQL)
4. Session manager setup
5. Repository instantiation
6. Service instantiation (dependency injection)
7. Event bus subscriber registration
8. HTTP server startup
9. Main server startup (IRC handlers, feed service, scheduler)

### Frontend Structure

React app with modern tooling:
- **Router**: TanStack Router with file-based routing
- **State Management**: TanStack Query for server state, React Ridge State for client state
- **Forms**: Formik with Zod validation
- **Styling**: Tailwind CSS 4 with custom theme
- **Build**: Vite with SWC

## Important Implementation Notes

### Indexer Definitions

Indexer definitions are loaded from:
- Embedded definitions (compiled into binary)
- Custom definitions via `AUTOBRR__CUSTOM_DEFINITIONS` env var or `config.toml`

Each definition contains:
- IRC announce patterns with regex capture groups
- API endpoints and authentication
- Parsing rules to map captures to Release fields

### Filter Matching Logic

Filters support:
- Regex and wildcard patterns (`*`, `!` for negation)
- Range matching (year, resolution, size)
- Advanced checks (uploader, smart episode, duplicate detection)
- External filters (exec scripts or webhooks that can approve/reject)

When a release matches a filter:
1. Size may be fetched via torrent download or API call if not in announce
2. External filters are executed if configured
3. All enabled actions are executed in sequence

### Database Migrations

Migrations are in `internal/database/migrations/`:
- Automatically applied on startup
- Both SQLite and PostgreSQL supported
- Use `squirrel` for query building

### Configuration

Config file: `config.toml`
- Supports dynamic reload (most settings apply without restart)
- All settings overridable via `AUTOBRR__*` environment variables
- Docker secrets support via `_FILE` suffix (e.g., `AUTOBRR__POSTGRES_PASSWORD_FILE`)

### Testing Strategy

- **Unit tests** - Test domain logic, services, parsing
- **Integration tests** - Test database operations, require PostgreSQL container and `-tags=integration`
- **No frontend tests** - Backend-focused testing strategy
- Tests run in CI with timezone UTC for stability

## Common Development Tasks

### Adding a New Download Client

1. Create client implementation in `internal/download_client/<client>/`
2. Implement required interface methods
3. Add client type to `domain/download_client.go`
4. Register in `internal/action/service.go`
5. Add UI components in `web/src/screens/settings/DownloadClient*.tsx`

### Adding a New Indexer

1. Create YAML definition in indexer definitions repo
2. Test with mock indexer: `go run test/mockindexer/main.go`
3. Add definition to autobrr/autobrr-indexers repo

### Modifying Filter Logic

Filter matching is in `internal/domain/filter.go`:
- `Filter.CheckFilter()` contains core matching logic
- Service layer (`internal/filter/service.go`) handles external checks (size, uploader, API calls)

### Working with IRC

IRC handlers are in `internal/irc/`:
- `service.go` - Manages multiple network handlers
- `handler.go` - Single network connection with reconnect logic
- `parser.go` - Message parsing and validation

### Debugging

1. Enable debug logging: `AUTOBRR__LOG_LEVEL=DEBUG`
2. View structured logs in terminal or web UI (Settings → Logs)
3. Use mock indexer for testing IRC announces: `go run test/mockindexer/main.go`
4. Metrics available at `http://localhost:9074/metrics` if enabled

## Git Workflow

- Main branch: `develop` (use for PRs)
- Release branch: `master`
- Commit convention: [Conventional Commits](https://www.conventionalcommits.org/)
  - Examples: `fix(indexers): improve parsing`, `feat(notifications): add service`
- Commits are squashed on merge

## Building for Production

### Cross-platform binaries

```bash
# Install GoReleaser
go install github.com/goreleaser/goreleaser/v2@latest

# Build all platforms
goreleaser build --snapshot --clean
```

### Docker

```bash
# Standard build
make build/docker

# Multi-platform (Apple ARM, etc.)
make build/dockerx
```

## Environment Variables

Key environment variables (full list in README.md):
- `AUTOBRR__HOST` - Listen address (default: `127.0.0.1`)
- `AUTOBRR__PORT` - Listen port (default: `7474`)
- `AUTOBRR__LOG_LEVEL` - Log level (DEBUG, INFO, WARN, ERROR)
- `AUTOBRR__DATABASE_TYPE` - Database type (`sqlite`, `postgres`)
- `AUTOBRR__POSTGRES_*` - PostgreSQL connection settings
- `AUTOBRR__OIDC_*` - OpenID Connect authentication settings

## Key Files to Understand

- `cmd/autobrr/main.go` - Application entry point and initialization
- `internal/server/server.go` - Main server orchestration
- `internal/domain/*.go` - Core domain models and business logic
- `internal/filter/service.go` - Filter matching orchestration
- `internal/action/service.go` - Action execution
- `internal/irc/handler.go` - IRC connection and message handling
- `Makefile` - Build commands and targets
