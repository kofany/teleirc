# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TeleIRC is a bidirectional Telegram ↔ IRC bridge written in Go. It relays messages, files, media, and channel events between a Telegram group and an IRC channel.

## Commands

**Build:**
```bash
./build_binary.sh
```

**Test:**
```bash
go test ./...
go test -coverprofile=c.out ./...  # with coverage
```

**Lint:**
```bash
golangci-lint run
```

**Docker build:**
```bash
bash deployments/container/build_image.sh
```

## Architecture

The application uses a **bridge pattern** with two independent clients communicating via callbacks:

```
main.go (orchestration, signal handling)
    ├── internal/handlers/irc/      IRC client (girc library)
    └── internal/handlers/telegram/ Telegram client (telegram-bot-api)
```

**Key design decisions:**
- IRC and Telegram clients are decoupled - they communicate only via `SendMessage` callbacks passed during initialization
- Heavy use of interfaces (`ClientInterface`, `DebugLogger`) for testability
- Handlers follow the pattern: `type Handler = func(c ClientInterface) func(*girc.Client, girc.Event)`
- Configuration loaded from environment variables via `caarlos0/env` (see `env.example`)

**Message flow:**
- IRC → Telegram: girc event → handler → format message → callback to Telegram client
- Telegram → IRC: Update → handler dispatcher → format message → callback to IRC client

## Code Conventions

- Function names should be platform-agnostic (not IRC-specific or Telegram-specific)
- Handlers named in camelCase: `joinHandler`, `messageHandler`
- Consistency across IRC and Telegram implementations where possible

## Testing

Uses `golang/mock` for mock generation. Test pattern:

```go
ctrl := gomock.NewController(t)
defer ctrl.Finish()
mockClient := NewMockClientInterface(ctrl)
mockClient.EXPECT().Logger().Return(mockLogger)
// assertions...
```

Tests verify behavior contracts, not implementation details. See `docs/dev/testing.md` for full guide.

## Key Dependencies

- `github.com/lrstanley/girc` - IRC protocol
- `github.com/go-telegram-bot-api/telegram-bot-api` - Telegram API
- `github.com/caarlos0/env/v6` - Environment config parsing
- `github.com/golang/mock` - Mock generation
- `modernc.org/sqlite` - Pure Go SQLite (for MediaShare)

## MediaShare Microservice

Self-hosted media sharing service for videos, voice messages, and photos from Telegram.

**Location:** `cmd/mediashare/` (standalone binary), `internal/mediashare/` (library)

**Build MediaShare:**
```bash
go build -o mediashare ./cmd/mediashare
```

**Run MediaShare:**
```bash
./mediashare -port 8080 -url https://media.example.com -key YOUR_API_KEY
```

**Architecture:**
```
internal/mediashare/
├── server.go      # HTTP server, routing, cleanup worker
├── storage.go     # File storage, ID generation
├── database.go    # SQLite metadata (username, timestamps, open count)
├── templates.go   # HTML templates (Tailwind CSS, dark/light mode)
└── i18n.go        # Translations (Polish/English)
```

**Key features:**
- 5-character alphanumeric URLs: `https://domain/Ab3xK`
- SQLite for metadata persistence
- Auto-cleanup: files not opened for 72h are deleted
- Dark/light mode with CSS `prefers-color-scheme`
- Multi-language support (pl/en)

**Configuration (env vars):**
| Variable | Default | Description |
|----------|---------|-------------|
| `MEDIASHARE_PORT` | 8080 | Server port |
| `MEDIASHARE_BASE_URL` | http://localhost:8080 | Public URL for links |
| `MEDIASHARE_API_KEY` | (none) | Authentication key |
| `MEDIASHARE_STORAGE_PATH` | ./uploads | File storage directory |
| `MEDIASHARE_DB_PATH` | ./mediashare.db | SQLite database path |
| `MEDIASHARE_RETENTION_HOURS` | 72 | Auto-delete after X hours |
| `MEDIASHARE_SERVICE_NAME` | MediaShare | Branding in HTML |
| `MEDIASHARE_LANGUAGE` | pl | UI language (pl/en) |

**TeleIRC integration:**
Set these in TeleIRC's env to enable MediaShare uploads:
```bash
MEDIASHARE_ENABLED=true
MEDIASHARE_ENDPOINT=http://localhost:8080
MEDIASHARE_API_KEY=your_shared_key
```

**Deployment guide:** See `docs/websetup.md` for Caddy/systemd configuration.
