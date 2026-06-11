# Go & HelloFresh Conventions — Reference

These are the Go and HelloFresh-specific rules the `pr-review-deep` skill applies as a FALLBACK. The priority order (per design Section 8) is:

1. Repo's own `CLAUDE.md`
2. Linter configs (`.golangci.yml`, `.eslintrc`, etc.)
3. Existing patterns in the same package/directory
4. Go conventions (Effective Go) — for Go files
5. **HelloFresh conventions (this file)** — fallback for HelloFresh repos

If a higher-priority rule disagrees with what's below, the higher-priority rule wins. This file is consulted only when the file under review is Go AND no higher-priority rule applies.

## Hexagonal architecture

Expected layout:
```
pkg/{domain}/
├── dependency/       # Adapters — external service clients
├── http/             # Adapters — HTTP handlers (validation, routing only)
├── service.go        # Core — business logic, orchestration, transactions
├── model.go          # Core — domain models and DTOs
└── *_test.go         # Tests for each layer
```

**Violations to flag (`issue (blocking)`):**

- Handler → Repository direct access (must go through service)
- Service → HTTP concerns (request/response handling)
- Domain models → Infrastructure dependencies (AWS SDK, DB drivers)
- Circular dependencies between packages

**The "input validation philosophy" — caller's responsibility:**

```go
// CORRECT — handler validates, service trusts
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", 400)
        return
    }
    if req.Email == "" || req.Name == "" {
        http.Error(w, "email and name required", 400)
        return
    }
    user, err := h.service.CreateUser(r.Context(), req.Email, req.Name)
    // ...
}

// Service trusts caller — NO redundant validation
func (s *Service) CreateUser(ctx context.Context, email, name string) (*User, error) {
    user := &User{Email: email, Name: name}
    return s.repo.Save(ctx, user)
}
```

**When to flag missing validation:** public API boundaries (HTTP/gRPC), external system inputs (queue messages, webhooks), user-provided data.

**When NOT to flag:** internal service methods (trust caller), methods where constructor validated dependencies, private functions within the same package.

## SOLID — interface design

**Consumer-side, unexported, focused interfaces:**

```go
// CORRECT
type customerGetter interface {
    FindCustomerInfoByToken(ctx context.Context, market hf.Market, token string) (*Customer, error)
}

type Service struct {
    customerGetter customerGetter
}

// WRONG — large exported interface with unnecessary methods
type CustomerRepository interface {
    FindCustomerInfoByToken(ctx context.Context, market hf.Market, token string) (*Customer, error)
    UpdateCustomer(ctx context.Context, customer *Customer) error
    DeleteCustomer(ctx context.Context, id string) error
    // ... 10 more methods the service doesn't need
}
```

**Constructor injection:**

```go
// CORRECT — deps injected, validated at construction
func NewService(
    customerGetter customerGetter,
    ordersGetter ordersGetter,
    logger *logrus.Logger,
) *Service {
    if customerGetter == nil {
        panic("customerGetter is required")
    }
    if ordersGetter == nil {
        panic("ordersGetter is required")
    }
    return &Service{
        customerGetter: customerGetter,
        ordersGetter:   ordersGetter,
        logger:         logger,
    }
}
```

**Methods can safely use dependencies without nil checks** — the constructor guarantees non-nil. Flagging nil checks on dep usage when the constructor validates is a 2-why-gate DROP.

## Go best practices

**Unused variables/constants:** Flag `issue (blocking)` for any unused `const`, `var`, or local variable except `_` for explicit interface compliance.

**Custom types — direct usage in SQL:**
```go
// CORRECT
var market hf.Market = "us"
_, err := db.ExecContext(ctx, query, market)

// WRONG — unnecessary conversion
_, err := db.ExecContext(ctx, query, string(market))
```

**Structured errors with context:**
```go
err := errors.New("intent classification failed").
    AddDetail("message", message).
    AddDetail("market", market)

if errors.IsNotFound(err) {
    // handle not found
}
```

**Graceful degradation only for non-critical paths:**
```go
// CORRECT for non-critical enrichment
func (s *Service) EnrichWithCustomerData(ctx context.Context, conv *Conversation) {
    customerData, err := s.customerClient.Get(ctx, conv.CustomerID)
    if err != nil {
        logger.FromContext(ctx).WithError(err).Warn("Failed to enrich, continuing")
        return
    }
    conv.CustomerData = customerData
}

// WRONG for critical paths (auth, payment, etc.)
func (s *Service) AuthenticateUser(ctx context.Context, token string) (*User, error) {
    user, err := s.authClient.Validate(token)
    if err != nil {
        logger.Warn("auth failed, continuing anyway")    // ← do not swallow
        return &User{}, nil
    }
    return user, nil
}
```

**Context-aware logging:**
```go
// CORRECT
logger.FromContext(ctx).WithFields(logrus.Fields{
    "conversation_id": conversationID,
    "intent":          intentName,
}).Info("Intent classified")

// WRONG — bypasses request tracing
logrus.Info("Intent classified")
```

**Comments on exported items:** all exported types, functions, methods MUST have doc comments. Unexported items don't require them (but can have them when complex).

## Testing conventions

**Coverage rule:** flag `issue (blocking)` only when a public method has NO unit tests AND NO integration tests covering it. Either is sufficient. Don't demand both.

**Mocks:** use mockery-generated mocks (`make mocks`). Manual mock structs are `issue (blocking)`.

**Table-driven test pattern (HelloFresh standard):**

```go
func TestService_ProcessIntent(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name       string
        input      InputType
        setupMocks func(*mocks)
        wantErr    error
        want       ExpectedType
    }{
        {
            name:  "successful intent classification",
            input: "where is my order",
            setupMocks: func(m *mocks) {
                m.bedrockClient.EXPECT().
                    Classify(mock.Anything, "where is my order").
                    Return(Intent{Name: "order_status"}, nil)
            },
            want:    Intent{Name: "order_status"},
            wantErr: nil,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            mocks := setupMocks(t)
            if tc.setupMocks != nil {
                tc.setupMocks(mocks)
            }
            subject := NewService(mocks.bedrockClient)

            got, gotErr := subject.ProcessIntent(context.Background(), tc.input)

            hfassert.ErrorInChain(t, tc.wantErr, gotErr)
            assert.Equal(t, tc.want, got)
        })
    }
}
```

**Variable naming in tests:**
- `tc` for test case in loops
- `got`, `gotErr` for method results
- `subject` for the instance being tested
- `wantErr`, `want` for expected values
- `mock` prefix for mock instances (`bedrockClientMock`)

If the repo uses different names (e.g., `tt` instead of `tc`), the repo wins per priority order.

## Security — flag immediately as `issue (blocking)`

- Hardcoded secrets (API keys, passwords, tokens)
- SQL injection via concatenation (use parameterized queries)
- Command injection via `exec.Command("sh", "-c", userInput + ...)`
- Logging sensitive data (passwords, credit cards, tokens, PII)
- Missing authentication/authorization on sensitive endpoints

## Performance — flag if performance-critical

- N+1 queries in loops (suggest batch operation)
- Unbounded goroutine spawn (suggest worker pool with semaphore)
- Unnecessary allocations in hot paths
- Missing context timeout on external API calls

For non-hot-path code, performance is `suggestion (non-blocking)` at most.

## What NOT to comment on

- Minor style/formatting (linter catches these — let it)
- Variable naming preferences when not truly confusing
- "Not how I would do it" but still correct
- Optimizations that don't matter for the current scale
- Validation already done elsewhere in the call chain
- Nil checks for dependencies validated in constructors
- Edge cases already covered by integration tests
- Issues already raised in previous comments and addressed
