+++
title = "How I Structure Go Projects in 2025"
date = 2025-07-14T09:00:00+05:30
description = "A practical project layout for Go services that keeps things simple as the codebase grows. Covers package organization, dependency injection, and testing patterns."
tags = ["go", "architecture", "backend"]
slug = "go-project-structure-2025"
draft = false
+++

Every six months someone publishes a new "standard Go project layout" that is more complex than the last. Most of them are over-engineered for the majority of Go projects.

Here is what I actually use for production services. It is not clever. It scales to medium-sized codebases (50k-100k lines) without becoming a burden.

## The Layout

```
myservice/
├── cmd/
│   └── myservice/
│       └── main.go              # entry point, wiring
├── internal/
│   ├── api/
│   │   ├── handler.go           # HTTP handlers
│   │   ├── middleware.go         # auth, logging, rate limiting
│   │   └── router.go            # route definitions
│   ├── domain/
│   │   ├── user.go              # domain types and interfaces
│   │   ├── order.go
│   │   └── errors.go            # domain-specific errors
│   ├── service/
│   │   ├── user_service.go      # business logic
│   │   └── order_service.go
│   ├── store/
│   │   ├── postgres/
│   │   │   ├── user_store.go    # PostgreSQL implementation
│   │   │   └── order_store.go
│   │   └── redis/
│   │       └── cache.go         # Redis cache implementation
│   └── config/
│       └── config.go            # env/file config loading
├── migrations/
│   ├── 001_create_users.sql
│   └── 002_create_orders.sql
├── go.mod
├── go.sum
├── Makefile
└── Dockerfile
```

That is it. No `pkg/`. No `adapters/`. No `ports/`. No hexagonal architecture diagrams. Just four packages under `internal/` that map to clear responsibilities.

## The Rules

### Rule 1: `internal/` for Everything

Using `internal/` means no other Go module can import your packages. This is intentional. Most services are not libraries. Nobody should be importing your HTTP handlers.

This frees you from worrying about API stability of internal packages. You can refactor freely.

### Rule 2: Domain Types Are Plain Structs

The `domain` package contains your core types and interfaces. No dependencies on databases, HTTP, or external libraries:

```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID        string
    Email     string
    Name      string
    CreatedAt time.Time
    UpdatedAt time.Time
}

type UserStore interface {
    Get(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

type UserService interface {
    Register(ctx context.Context, email, name, password string) (*User, error)
    Authenticate(ctx context.Context, email, password string) (*User, error)
    UpdateProfile(ctx context.Context, id, name string) (*User, error)
}
```

Interfaces are defined where they are **used**, not where they are implemented. The `domain` package defines what a `UserStore` looks like because the service layer depends on it. The `store/postgres` package just satisfies the interface.

### Rule 3: Business Logic in Services

The `service` package contains your actual business rules. It depends on domain interfaces, not concrete implementations:

```go
// internal/service/user_service.go
package service

import (
    "context"
    "fmt"
    
    "myservice/internal/domain"
)

type UserService struct {
    users  domain.UserStore
    hasher PasswordHasher
}

func NewUserService(users domain.UserStore, hasher PasswordHasher) *UserService {
    return &UserService{users: users, hasher: hasher}
}

func (s *UserService) Register(ctx context.Context, email, name, password string) (*domain.User, error) {
    // Check if email is already taken
    existing, err := s.users.GetByEmail(ctx, email)
    if err != nil && !errors.Is(err, domain.ErrNotFound) {
        return nil, fmt.Errorf("checking existing user: %w", err)
    }
    if existing != nil {
        return nil, domain.ErrEmailTaken
    }
    
    // Hash password
    hashed, err := s.hasher.Hash(password)
    if err != nil {
        return nil, fmt.Errorf("hashing password: %w", err)
    }
    
    user := &domain.User{
        ID:    generateID(),
        Email: email,
        Name:  name,
    }
    
    if err := s.users.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("creating user: %w", err)
    }
    
    return user, nil
}
```

Notice: no HTTP concepts here. No request parsing, no response formatting. Just business rules.

### Rule 4: Handlers Are Thin

HTTP handlers parse the request, call a service method, and format the response. Three steps, no business logic:

```go
// internal/api/handler.go
package api

import (
    "encoding/json"
    "net/http"
    
    "myservice/internal/domain"
)

type Handler struct {
    users domain.UserService
}

func (h *Handler) RegisterUser(w http.ResponseWriter, r *http.Request) {
    // 1. Parse request
    var req struct {
        Email    string `json:"email"`
        Name     string `json:"name"`
        Password string `json:"password"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }
    
    // 2. Call service
    user, err := h.users.Register(r.Context(), req.Email, req.Name, req.Password)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailTaken):
            writeError(w, http.StatusConflict, "email already registered")
        default:
            writeError(w, http.StatusInternalServerError, "internal error")
        }
        return
    }
    
    // 3. Write response
    writeJSON(w, http.StatusCreated, user)
}
```

If a handler function is longer than 30 lines, something is wrong. Either it is doing validation that belongs in the service layer, or it is doing data transformation that should be a separate function.

### Rule 5: Wiring Happens in main.go

Dependency injection in Go does not need a framework. Constructor functions and `main.go` are enough:

```go
// cmd/myservice/main.go
package main

import (
    "log"
    "net/http"
    
    "myservice/internal/api"
    "myservice/internal/config"
    "myservice/internal/service"
    "myservice/internal/store/postgres"
    "myservice/internal/store/redis"
)

func main() {
    cfg := config.Load()
    
    // Initialize stores
    db := postgres.Connect(cfg.DatabaseURL)
    cache := redis.Connect(cfg.RedisURL)
    
    userStore := postgres.NewUserStore(db)
    cachedUserStore := redis.NewCachedUserStore(cache, userStore)
    
    // Initialize services
    userService := service.NewUserService(cachedUserStore, service.NewBcryptHasher())
    
    // Initialize API
    handler := api.NewHandler(userService)
    router := api.NewRouter(handler)
    
    log.Printf("starting server on %s", cfg.Port)
    log.Fatal(http.ListenAndServe(":"+cfg.Port, router))
}
```

All the wiring is in one place. You can read `main.go` and understand every dependency in the system. No magic, no reflection, no container.

## Testing

This structure makes testing straightforward because every layer depends on interfaces.

### Unit Testing Services

```go
func TestUserService_Register(t *testing.T) {
    store := &mockUserStore{} // implements domain.UserStore
    hasher := &mockHasher{hash: "hashed-password"}
    
    svc := service.NewUserService(store, hasher)
    
    user, err := svc.Register(context.Background(), "test@example.com", "Test User", "password123")
    
    assert.NoError(t, err)
    assert.Equal(t, "test@example.com", user.Email)
    assert.Equal(t, "Test User", user.Name)
    assert.True(t, store.createCalled)
}

func TestUserService_Register_DuplicateEmail(t *testing.T) {
    store := &mockUserStore{
        existingUser: &domain.User{Email: "test@example.com"},
    }
    hasher := &mockHasher{}
    
    svc := service.NewUserService(store, hasher)
    
    _, err := svc.Register(context.Background(), "test@example.com", "Test", "pass")
    
    assert.ErrorIs(t, err, domain.ErrEmailTaken)
}
```

### Integration Testing with Real Databases

For store tests, I use testcontainers to spin up real PostgreSQL:

```go
func TestPostgresUserStore(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    ctx := context.Background()
    container := startPostgres(t) // uses testcontainers
    db := connectDB(t, container)
    runMigrations(t, db)
    
    store := postgres.NewUserStore(db)
    
    // Test the real implementation against a real database
    user := &domain.User{
        ID:    "test-id",
        Email: "test@example.com",
        Name:  "Test User",
    }
    
    err := store.Create(ctx, user)
    assert.NoError(t, err)
    
    got, err := store.Get(ctx, "test-id")
    assert.NoError(t, err)
    assert.Equal(t, user.Email, got.Email)
}
```

## What I Intentionally Skip

- **`pkg/` directory** -- I do not build shared libraries within services. If code needs to be shared, it becomes its own module
- **Hexagonal/Clean/Onion architecture** -- The concept is right (depend on abstractions), but the naming and ceremony are overkill for most Go projects
- **Wire or other DI frameworks** -- Constructor functions work fine up to a surprising scale
- **Interface for everything** -- Only define interfaces at package boundaries where you need substitutability. Do not interface-wrap your config loader

## When to Restructure

This layout works until it does not. Signs you need to evolve:

- **A package has more than 15-20 files** -- split it into sub-packages by subdomain
- **Circular dependency** -- you have a layering violation, usually service-to-service calls that should go through domain events
- **Multiple teams working in the same service** -- time to split into separate services, not reorganize packages

The goal is always the same: make the code easy to change. Package structure is a tool for managing change, not an end in itself.
