---
name: golang
description: "Use when working on projects involving Go code."
---

# Go Development Skill

## References

- [The Go Programming Language Specification](https://go.dev/ref/spec)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Go Standard Library](https://pkg.go.dev/std)
- [Go Module Reference](https://go.dev/ref/mod)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)

## Code Quality Checks

Run these in order after every change:

```bash
# 1. Format code
gofumpt -w .
goimports -w .

# 2. Check compilation
go build ./...

# 3. Lint (strict — use repo .golangci.yml; for new projects enable all linters, no broad excludes)
golangci-lint run --timeout=5m ./...

# 4. Run tests with race detection
go test -race ./...

# 5. Security vulnerability check (MANDATORY)
govulncheck ./...
```

## Project Structure

For new projects, use this canonical layout:

```
project-root/
├── cmd/
│   └── app/
│       └── main.go
├── internal/
│   └── domain/
├── pkg/           # Only if sharing across projects
├── tests/
├── go.mod
├── go.sum
└── README.md
```

- `cmd/` — Entry points. Each subdirectory is a binary.
- `internal/` — Private application code. Enforced by the compiler.
- `pkg/` — Public library code. Only create if genuinely shared externally.
- `tests/` — All tests for new projects. Place tests here as black-box tests
  importing packages.

For existing projects, follow whatever layout and test placement is already in
place.

## Testing

**Always check the project for an existing test framework first.**

For new projects, use `testify` (`assert` and `require` subpackages).

### Requirements

- Every new function must have a corresponding test.
- Use table-driven tests for functions with multiple input/output combinations.
- Use `t.Parallel()` where safe.
- Run with race detection: `go test -race ./...`
- Use `t.Helper()` in test helper functions.
- Use `testify/require` for fatal assertions, `testify/assert` for non-fatal.

### Commands

```bash
# Run all tests
go test -race ./...

# Run tests with verbose output
go test -race -v ./...

# Run specific test
go test -race -run TestFunctionName ./path/to/package

# Run tests with coverage
go test -race -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run benchmarks
go test -bench=. -benchmem ./...
```

### Table-Driven Test Example

```go
func TestAdd(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            assert.Equal(t, tt.expected, Add(tt.a, tt.b))
        })
    }
}
```

## Naming Conventions

| Item              | Convention                                      | Example                        |
| ----------------- | ----------------------------------------------- | ------------------------------ |
| Packages          | lowercase, single word                          | `http`, `user`                 |
| Files             | snake_case                                      | `user_service.go`              |
| Types/Interfaces  | PascalCase                                      | `UserService`, `Reader`        |
| Functions/Methods | PascalCase (exported), camelCase (unexported)   | `GetUser()`, `parseInput()`    |
| Constants         | PascalCase or SCREAMING_SNAKE for special cases | `MaxRetries`, `DefaultTimeout` |
| Variables         | camelCase                                       | `itemCount`                    |
| Acronyms          | all caps                                        | `HTTPClient`, `userID`         |
| Test files        | `*_test.go`                                     | `user_service_test.go`         |
| Test functions    | `TestXxx`                                       | `TestGetUser`                  |
| Benchmarks        | `BenchmarkXxx`                                  | `BenchmarkGetUser`             |

## Idioms

### Accept Interfaces, Return Structs

```go
func NewService(repo UserRepository) *Service {
    return &Service{repo: repo}
}
```

### Small Interfaces

Prefer 1-2 method interfaces. Compose larger interfaces from smaller ones.

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type ReadWriter interface { Reader; Writer }
```

### context.Context as First Parameter

```go
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
```

### Error Wrapping

```go
if err != nil {
    return fmt.Errorf("fetching user %s: %w", id, err)
}
```

### Functional Options Pattern

```go
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### Dependency Injection via Constructors

```go
func NewOrderService(repo OrderRepository, notifier Notifier) *OrderService {
    return &OrderService{repo: repo, notifier: notifier}
}
```

### Defer for Cleanup

```go
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
```

### Channel Direction Constraints

```go
func producer(ch chan<- int) { }
func consumer(ch <-chan int) { }
```

### Range Over Int (1.22+)

```go
for i := range 10 {
    fmt.Println(i)
}
```

### Iterator Patterns (1.23+)

Use the `iter` package for custom iterators with `iter.Seq` and `iter.Seq2`.

### Avoid Package-Level State

Pass dependencies explicitly through constructors and function parameters.

## Design Patterns

### Middleware (HTTP)

```go
func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

### Worker Pool

```go
func WorkerPool(ctx context.Context, jobs <-chan Job, workers int) <-chan Result {
    results := make(chan Result)
    g, ctx := errgroup.WithContext(ctx)
    for range workers {
        g.Go(func() error {
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                case results <- process(job):
                }
            }
            return nil
        })
    }
    go func() {
        g.Wait()
        close(results)
    }()
    return results
}
```

### Fan-Out/Fan-In

Distribute work across goroutines (fan-out), collect results into one channel
(fan-in).

### Pipeline

Chain stages connected by channels. Each stage is a goroutine reading from
input, processing, writing to output.

### Circuit Breaker

Track failures; open circuit after threshold; allow periodic retries.

### Strategy via Interfaces

Define behavior as an interface, inject the desired implementation at
construction.

## Anti-Patterns

| Anti-Pattern                         | Why It's Bad                     | Do Instead                          |
| ------------------------------------ | -------------------------------- | ----------------------------------- |
| Naked goroutines                     | Leak, panic unrecovered          | Use `errgroup` or managed lifecycle |
| Ignoring errors (`_ = fn()`)         | Hides bugs                       | Handle every error explicitly       |
| Interface pollution                  | Unnecessary abstraction          | Define interfaces at the consumer   |
| Package stutter (`user.UserService`) | Redundant                        | `user.Service`                      |
| `init()` abuse                       | Hidden side effects, test issues | Explicit initialization in `main`   |
| Global state / singletons            | Tight coupling, untestable       | Dependency injection                |
| Channels for simple shared state     | Overcomplicated                  | `sync.Mutex`                        |
| `panic` in library code              | Crashes callers                  | Return errors                       |
| Empty interface (`any`) overuse      | Loses type safety                | Use generics or concrete types      |

## Error Handling

- Always handle errors explicitly. Never discard without documented
  justification.
- Wrap errors with context:
  ```go
  return fmt.Errorf("saving order %d: %w", id, err)
  ```
- Check errors with `errors.Is` and `errors.As`:
  ```go
  if errors.Is(err, ErrNotFound) { ... }
  ```
- Define sentinel errors per package:
  ```go
  var ErrNotFound = errors.New("not found")
  ```
- Define custom error types for rich context:
  ```go
  type ValidationError struct {
      Field   string
      Message string
  }
  func (e *ValidationError) Error() string {
      return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
  }
  ```
- Never `panic` in library code. Reserve `panic` for truly unrecoverable states
  in `main`.

## Concurrency

- **Always pass `context.Context`** as the first parameter.
- **Use `errgroup`** for managed goroutine lifecycles:
  ```go
  g, ctx := errgroup.WithContext(ctx)
  g.Go(func() error { return doWork(ctx) })
  if err := g.Wait(); err != nil { ... }
  ```
- **Prefer `sync.Mutex`** for simple shared state over channels.
- **Constrain channel direction** in function signatures.
- **Use `sync.Once`** for one-time initialization.
- **Use `sync.Pool`** for frequently allocated/freed objects.
- **Handle context cancellation** in long-running goroutines:
  ```go
  select {
  case <-ctx.Done():
      return ctx.Err()
  case result := <-ch:
      // process
  }
  ```
- **Race detector**: always run `go test -race` in CI and locally.
- **Never start a goroutine without a clear shutdown path.**

## Documentation

- All exported types, functions, methods, and constants must have doc comments.
- Doc comments start with the name of the item:
  ```go
  // Service manages user operations.
  type Service struct { ... }

  // GetUser retrieves a user by ID.
  func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
  ```
- Package comments go in `doc.go` or at the top of the primary file.
- Use `go doc` and `pkgsite` to verify documentation renders correctly.

## Dependencies

**Always check what the project already uses before adding anything.**

For new projects:

| Purpose        | Recommended                   | Notes                            |
| -------------- | ----------------------------- | -------------------------------- |
| Logging        | `go.uber.org/zap`             | Structured, high-performance     |
| Testing        | `github.com/stretchr/testify` | `assert` and `require`           |
| HTTP framework | `net/http` (stdlib)           | 1.22+ enhanced routing           |
| Hot reload     | `github.com/air-verse/air`    | Dev only                         |
| Linting        | `golangci-lint`               | External tool, not a dependency  |
| Formatting     | `gofumpt`, `goimports`        | External tools                   |
| Security       | `govulncheck`                 | External tool, MANDATORY         |
| Goroutine mgmt | `golang.org/x/sync/errgroup`  | Always use over naked goroutines |
| Config         | `github.com/spf13/viper`      | Or `envconfig` for simple needs  |
| CLI            | `github.com/spf13/cobra`      | For CLI applications             |

Target Go 1.23+ as minimum. Prefer latest stable release.

## Performance Considerations

- **Profile before optimizing**: use `pprof` for CPU, memory, and goroutine
  profiling.
- **Benchmark**: write `Benchmark*` functions; run with `-benchmem`.
- **Preallocate slices** when length is known: `make([]T, 0, n)`.
- **Use `strings.Builder`** for string concatenation in loops.
- **Use `sync.Pool`** for high-frequency allocations.
- **Avoid premature optimization** — measure first.
- **Use pointer receivers** for large structs; value receivers for small
  immutable types.
- **Minimize allocations** in hot paths. Check with `go test -benchmem`.

## Design Principles

- **KISS** — Keep it simple.
- **DRY** — Don't repeat yourself, but a little copying is better than a little
  dependency.
- **YAGNI** — Don't build what you don't need yet.
- **SOLID** — Apply where it improves clarity, not dogmatically.
- **Composition over inheritance** — Embed types, compose interfaces.
- **Accept interfaces, return structs** — Maximize flexibility for callers.
- **Make the zero value useful** — Design types that work without explicit
  initialization.
- **Clear is better than clever** — Readability over cleverness.
- **Package names describe what they provide** — Not what they contain.
- **Errors are values** — Handle them as part of the normal control flow.

## Checklist Before Completion

- [ ] Code compiles: `go build ./...`
- [ ] Code is formatted: `gofumpt -w .` and `goimports -w .`
- [ ] Code is linted: `golangci-lint run ./...`
- [ ] All tests pass: `go test -race ./...`
- [ ] Security check: `govulncheck ./...`
- [ ] All exported items have doc comments
- [ ] Every new function has a corresponding test
- [ ] Errors are wrapped with context
- [ ] No naked goroutines
- [ ] No ignored errors without justification
- [ ] Interfaces are defined by the consumer, not the provider
