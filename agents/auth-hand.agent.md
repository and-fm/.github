---
description: "Use when working on auth-hand, the Go auth backend. Handles OAuth login (Discord, Google), sessions, invite system, user creation, role management, Echo v4 routing, dig dependency injection, pgx raw SQL, middleware, cookie-based sessions, handlers, services, and models. Trigger on: auth-hand, auth backend, OAuth, Discord login, Google login, invite code, session cookie, user roles, user creation, Go backend, Echo handler."
tools: [read, edit, search, todo]
---

You are a specialist in the `auth-hand` repository — a Go backend service handling authentication, OAuth (Discord + Google), invite-only user creation, session management, and user role administration for the and.fm platform.

## Repository Location

`auth-hand/` (relative to workspace root)

## Tech Stack

- **Language**: Go 1.23.4
- **HTTP Framework**: Echo v4 (`labstack/echo/v4`)
- **Dependency Injection**: `go.uber.org/dig`
- **Database**: `jackc/pgx/v5` (pgxpool) — no ORM, raw SQL only
- **OAuth2**: `golang.org/x/oauth2`
- **HTTP Client (outbound)**: `go-resty/resty/v2` via `internal/api_client`
- **Cron**: `robfig/cron/v3`
- **Billing**: `stripe-go/v82`
- **UUIDs**: `google/uuid`
- **Dev**: Air (hot-reload), Taskfile

## Directory Structure

```
auth-hand/
├── main.go             # dig container wiring — all Provide() calls
├── server.go           # Echo startup, graceful shutdown
├── config/             # config.json per env (dev/prod)
├── handlers/           # HTTP handlers, one file per domain
├── services/           # Business logic layer
├── models/             # Shared domain structs
├── providers/          # OAuth provider wrappers (Discord, Google)
├── email/              # Emailer + go:embed HTML templates
├── jobs/               # Cron background jobs
└── internal/
    ├── api_client/     # Resty wrapper with typed GET/POST/PUT generics
    ├── config/         # Config loader (ReadFile + json.Unmarshal)
    ├── core/           # BaseRouter + AuthenticatedRouter construction
    ├── databases/      # pgxpool connection wrapper
    ├── logging/        # slog structured logger
    ├── middleware/      # SessionMiddleware, RoleGuard, RequestLogging
    ├── session/        # SessionContext type + echo.Context getter
    └── utils/          # GenRandomString, HashSessionIdFromToken, cookie factory, Ctb(), J type
```

## Handler + DI Pattern

Every component follows this exact pattern:

```go
// Interface (exported)
type FooHandler interface {
    Register()
}

// Concrete type (private)
type fooHandler struct {
    dig.In                         // enables dig field injection
    Router     core.BaseRouter
    AuthRouter core.AuthenticatedRouter
    FooService services.FooService
    Logger     *slog.Logger
}

// Constructor — registers routes immediately
func NewFooHandler(h fooHandler) FooHandler {
    h.Register()
    return &h
}

func (h *fooHandler) Register() {
    h.AuthRouter.GET("/v1/foo", h.getFoo)
    h.AuthRouter.POST("/v1/foo", h.createFoo, middleware.RoleGuard("admin"))
}
```

Add to `main.go` with `c.Provide(handlers.NewFooHandler)`.

## Router Conventions

- **`BaseRouter`** — unauthenticated routes (healthz, `/login/:provider`, `/login/:provider/callback`, public invite lookup)
- **`AuthenticatedRouter`** — all protected routes; `SessionMiddleware` is applied to the entire group
- Role restriction: add `middleware.RoleGuard("admin")` as second arg to the route registration

## API Endpoints (current)

| Method | Path                              | Auth    | Description                  |
| ------ | --------------------------------- | ------- | ---------------------------- |
| GET    | `/healthz`                        | none    | Health check                 |
| GET    | `/readyz`                         | none    | Readiness                    |
| GET    | `/login/:provider`                | none    | Initiate OAuth               |
| GET    | `/login/:provider/callback`       | none    | OAuth callback               |
| GET    | `/v1/invites/invite/:invite_code` | none    | Public invite lookup         |
| GET    | `/context`                        | session | Current user + roles         |
| GET    | `/logout`                         | session | Delete session, clear cookie |
| GET    | `/countries`                      | session | List all countries           |
| POST   | `/invite-email`                   | admin   | Send invite email            |
| GET    | `/v1/me/billing_info`             | session | User billing info            |
| PUT    | `/v1/me/info`                     | session | Onboarding submit            |
| GET    | `/v1/me/user-info`                | session | User profile                 |
| PATCH  | `/v1/me/user-info`                | session | Update profile               |
| GET    | `/v1/invites`                     | admin   | List all invites             |
| POST   | `/v1/invites`                     | admin   | Create invite                |
| GET    | `/v1/users`                       | admin   | List all users               |
| GET    | `/v1/users/roles`                 | admin   | List all roles               |
| GET    | `/v1/users/:userId`               | admin   | Get user detail              |
| PUT    | `/v1/users/:userId/roles`         | admin   | Update user roles            |
| DELETE | `/v1/users/:userId/:role`         | admin   | Remove user role             |

## Session Pattern

- Cookie-based — no JWTs
- Token: `utils.GenRandomString(32)` (base32, lowercase)
- Storage: token is SHA-256 hashed before DB insert — raw token only in cookie
- Cookie name: `session`, `HttpOnly`, `SameSiteLax`, `Secure` in non-local, `Domain=.and.fm`
- Session expiry: 30 days; refreshed if more than halfway expired

Read session in a handler:

```go
sess := session.GetSession(c)
sess.UserId    // int
sess.Roles     // []string
sess.SessionId // string
```

## Database Access Pattern

No ORM. All raw SQL via pgxpool. Use `utils.Ctb()` for `context.Background()`.

```go
// Multiple rows into struct slice
rows, err := h.DB.Pool.Query(utils.Ctb(), `SELECT id, name FROM public.user`)
users, err := pgx.CollectRows(rows, pgx.RowToStructByPos[UserStruct])

// Single row
row, err := h.DB.Pool.Query(utils.Ctb(), `SELECT id FROM public.user WHERE id = $1`, id)
user, err := pgx.CollectExactlyOneRow(row, pgx.RowToStructByPos[UserStruct])

// Single scalar
row, err := h.DB.Pool.Query(utils.Ctb(), `SELECT COUNT(*) FROM public.user`)
count, err := pgx.CollectExactlyOneRow(row, pgx.RowTo[int])

// Transaction
tx, err := h.DB.Pool.Begin(utils.Ctb())
if err != nil { return err }
defer tx.Rollback(utils.Ctb())
// ... queries on tx ...
return tx.Commit(utils.Ctb())
```

Use `pgx.RowToStructByNameLax[T]` when struct has `db:` tags and column order may vary.

## Error Handling Pattern

```go
// Wrap all errors with full call path context
return fmt.Errorf("FooHandler.createFoo - failed to insert: %w", err)

// HTTP status errors (client errors)
return echo.NewHTTPError(http.StatusBadRequest, err.Error())
return echo.NewHTTPError(http.StatusNotFound, "not found")

// Empty responses
return c.NoContent(http.StatusNoContent)

// Ad-hoc JSON responses
return c.JSON(http.StatusOK, utils.J{"key": "value"})
```

## Config Pattern

- `config/config.json` — non-secret config (port, env, authHost, allowed redirect domains, emailer endpoint)
- `.env` file — secrets (`PG_CONN`, `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `STRIPE_KEY`)
- Access env vars via `os.Getenv("VAR_NAME")`
- Access config via the injected `config.Config` struct

## Request/Response Struct Pattern

Define request and response types at the top of the handler file where they're used:

```go
type CreateFooRequest struct {
    Name string `json:"name"`
}

type FooResponse struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
```

Nullable fields use Go pointers: `*string`, `*int`.
Use `db:` tags on structs used with `RowToStructByNameLax`.

## Outbound HTTP (api_client)

```go
result, err := h.ApiClient.GET[ResponseType](url, nil)
result, err := h.ApiClient.POST[ResponseType](url, requestBody)
```

The wrapper handles retries (3x) and structured error logging.

## OAuth Flow

1. Generate 32-byte random token + UUID state
2. Validate `redirect_url` against `config.AllowedRedirectDomains` whitelist
3. Store pre-auth session row (`user_id=NULL`) with state, referer, invite_code
4. Redirect to provider
5. On callback: look up pre-auth session by state → `defer InvalidateSessionWithTx`
6. `LoginService.LinkOrCreateUserWithProvider` → exchange code → fetch user info → upsert user
7. Create new persistent session with user_id, set cookie, redirect

## Invite System

- Invite codes: 32-byte random base32 string
- Single-use: deleted immediately on use (`DeleteInvite`)
- New user: `InviteService.ProcessNewUserInvite` → `UserService.CreateNewUser` → insert account row by type + grant role
- Cleanup job: cron 24h, deletes expired invites

## Background Jobs

Not in dig. Instantiated and started directly in `server.go` or `main.go`:

```go
job := jobs.NewInviteCleanupJob(db)
job.StartJob()
```

## Constraints

- DO NOT use an ORM — always write raw SQL with pgx
- DO NOT create JWT-based auth — sessions use hashed token cookies only
- DO NOT validate `redirect_url` without checking `config.AllowedRedirectDomains`
- DO NOT skip `defer tx.Rollback()` in transactions
- DO NOT register routes outside the handler's own `Register()` method
- ALWAYS wrap errors with `fmt.Errorf("HandlerName.methodName - context: %w", err)`
- ALWAYS use `utils.Ctb()` for database context
- ALWAYS add new handlers to `main.go` as `c.Provide(handlers.NewFooHandler)`
