# CLAUDE.md - Development Guide for chproxy

## Project Overview

**chproxy** is a high-performance HTTP proxy and load balancer for ClickHouse databases. It provides sophisticated features like:

- **Intelligent Load Balancing**: Least-loaded + round-robin across multiple ClickHouse replicas
- **Caching Layer**: Concurrent request deduplication with Redis and local cache support
- **Security**: User authentication, IP-based ACLs, query execution limits
- **Observability**: 50+ Prometheus metrics for monitoring
- **High Availability**: Automatic failover, graceful config reload (SIGHUP)
- **Rate Limiting**: Per-user query queuing and rate limits

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                         chproxy                              │
├─────────────────┬──────────────┬────────────┬───────────────┤
│   HTTP Server   │    Proxy     │   Cache    │   Metrics     │
│   (main.go)     │  (proxy.go)  │ (cache/)   │ (metrics.go)  │
├─────────────────┴──────────────┴────────────┴───────────────┤
│  Config (config/)  │  Scope (scope.go)  │  Utils (utils.go)│
├────────────────────┴────────────────────┴───────────────────┤
│         Internal: topology, heartbeat, counter              │
└─────────────────────────────────────────────────────────────┘
```

### Key Files

- **main.go**: HTTP server, TLS, config loading, signal handling
- **proxy.go**: Core reverse proxy logic, request routing, retries
- **scope.go**: User/cluster management, rate limiting, query queueing
- **cache/**: Response caching with concurrent request deduplication
- **config/**: YAML configuration parsing and validation
- **internal/topology**: Node health checking and replica selection
- **internal/heartbeat**: Active health monitoring for ClickHouse nodes

## Development Philosophy

### Code Quality Standards

1. **Error Handling First**: Never panic in production code, always return errors
2. **Security by Default**: Validate all inputs, fail closed, document security boundaries
3. **Performance Matters**: This is a critical infrastructure component
4. **Observable**: Every important operation should have metrics
5. **Testable**: Write tests before fixing bugs, maintain >70% coverage
6. **Simple Over Clever**: Prefer readable code over premature optimization

### Critical Patterns

#### ✅ DO:
```go
// Return errors, never panic
func processRequest(r *http.Request) error {
    if err := validate(r); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    return nil
}

// Check all errors
body, err := io.ReadAll(r.Body)
if err != nil {
    return fmt.Errorf("failed to read body: %w", err)
}

// Use context for cancellation
ctx, cancel := context.WithTimeout(r.Context(), timeout)
defer cancel()
```

#### ❌ DON'T:
```go
// Never panic in production paths
if err != nil {
    panic(fmt.Sprintf("unexpected error: %s", err))
}

// Don't ignore errors
io.Copy(dst, src) // Missing error check!

// Don't use global state without protection
var globalClient = &http.Client{} // Race condition!
```

## Known Issues & Improvement Areas

### 🚨 Critical (Must Fix)

1. **Panic Statements**: 8 locations need conversion to error returns
   - main.go:71, 180
   - config/config.go:96
   - io.go:123
   - proxy.go:350, 362
   - config/types.go:127
   - internal/heartbeat/heartbeat.go:82

2. **Unchecked Errors**: 6+ critical cases
   - utils.go:105 - io.Copy error ignored
   - scope.go:343 - io.ReadAll error ignored
   - cache/redis_cache.go:160 - weak error handling
   - cache/tmp_file_response_writer.go:47, 54 - file close errors

3. **Security Gaps**
   - Auth precedence undefined (headers vs Basic Auth vs URL params)
   - IP spoofing possible via X-Forwarded-For
   - No input size validation (OOM risk)
   - Global http.DefaultClient used

### 🔧 High Priority (Should Fix)

1. **High Complexity Functions**
   - proxy.go:377 serveFromCache (153 lines, complexity ~12-15)
   - scope.go:104 incQueued (98 lines, complex nesting)
   - utils.go:197 skipLeadingComments (42 lines nested logic)

2. **Performance Issues**
   - Polling loops with 100ms sleeps
   - Double body reading for snippets
   - Unnecessary deepcopy per request for wildcard users

3. **Documentation**
   - Missing godoc comments on exported functions
   - No inline explanations for complex algorithms
   - Auth flow not documented

### 📝 Good to Have

1. Extract magic numbers to constants
2. Add comprehensive examples
3. Benchmark critical paths
4. Expand test coverage to 80%+

## Testing Strategy

### Running Tests
```bash
# All tests
make test

# With race detector
go test -race ./...

# Specific package
go test -v ./cache

# Coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Test Organization
- `*_test.go`: Unit tests alongside source
- `testdata/`: Test configuration files
- Use `github.com/stretchr/testify` for assertions

## Linting & Code Quality

### Running Linters
```bash
# Full lint check
make lint

# Or directly
golangci-lint run

# Auto-fix where possible
golangci-lint run --fix
```

### Enabled Linters (54 total)
See `.golangci.yml` for complete list. Key ones:
- **gosec**: Security issues
- **errcheck**: Unchecked errors
- **gocritic**: Performance and style
- **cyclop**: Cyclomatic complexity
- **staticcheck**: Static analysis

## Building & Deployment

### Local Development
```bash
# Build
make build

# Run with test config
make run

# Full release build
make release
```

### Docker
```bash
# Build image
docker build -t chproxy:latest .

# Run
docker run -p 8080:8080 -v $(pwd)/config.yml:/config.yml chproxy -config=/config.yml
```

## Configuration

chproxy uses YAML configuration. Key sections:

- **server**: HTTP/HTTPS listeners, timeouts
- **users**: Authentication and rate limits
- **clusters**: ClickHouse backend configuration
- **caches**: Response caching rules

See `examples/` directory for complete examples.

## Metrics & Monitoring

Exposed at `/metrics` endpoint (Prometheus format):

- **Request metrics**: Total, duration, errors
- **Cache metrics**: Hits, misses, size
- **Backend metrics**: Active connections, failures
- **Queue metrics**: Queued/rejected requests

## Security Considerations

1. **Never expose chproxy directly**: Always use a reverse proxy (nginx, Cloudflare)
2. **Use X-Forwarded-For carefully**: Configure `proxy.forwarded_for` header
3. **Enable HTTPS**: Use Let's Encrypt autocert or provide certificates
4. **Restrict /metrics**: Use `allowed_networks` for metrics endpoint
5. **Set query limits**: Configure `max_execution_time` and `max_queue_size`

## Contributing Guidelines

1. **Before Starting**
   - Check existing issues/PRs
   - Discuss major changes first
   - Read CONTRIBUTING.md

2. **Development Process**
   - Create feature branch from master
   - Write tests first (TDD)
   - Run linters before commit
   - Ensure all tests pass
   - Update documentation

3. **Code Review Checklist**
   - [ ] No panics in production code
   - [ ] All errors checked
   - [ ] Tests added/updated
   - [ ] Linters pass
   - [ ] Documentation updated
   - [ ] No security issues
   - [ ] Performance considered

## Performance Optimization

### Profiling
```bash
# CPU profile
go test -cpuprofile=cpu.prof -bench=.
go tool pprof cpu.prof

# Memory profile
go test -memprofile=mem.prof -bench=.
go tool pprof mem.prof
```

### Known Bottlenecks
1. **Request copying**: Use buffer pools where possible
2. **Config reload**: Can cause brief latency spike
3. **Cache eviction**: Monitor cache size

## Debugging

### Enable Debug Logging
Set `log_debug: true` in config.

### Common Issues
1. **"No available replicas"**: Check heartbeat, ClickHouse health
2. **"Queue is full"**: Increase `max_queue_size` or add capacity
3. **"Cache errors"**: Check Redis connectivity, disk space

## Resources

- **Official Docs**: https://www.chproxy.org/
- **ClickHouse Docs**: https://clickhouse.com/docs
- **Go Best Practices**: https://go.dev/doc/effective_go
- **Security**: https://owasp.org/www-project-go-secure-coding-practices-guide/

## Architecture Decisions

### Why atomic.Value for config?
Allows lock-free reads during hot path while supporting graceful config updates.

### Why custom caching?
ClickHouse queries are expensive. Concurrent request deduplication prevents thundering herd.

### Why not use httputil.ReverseProxy?
Need fine-grained control over retries, failover, caching, and metrics.

## Maintenance

### Regular Tasks
- [ ] Update dependencies monthly (`go get -u ./...`)
- [ ] Review security advisories
- [ ] Update golangci-lint version
- [ ] Check for Go version updates
- [ ] Review and clean up metrics

### Versioning
Uses semantic versioning (vX.Y.Z):
- **X**: Breaking changes
- **Y**: New features
- **Z**: Bug fixes

---

## Quick Start for New Developers

1. **Clone and build**
   ```bash
   git clone https://github.com/ContentSquare/chproxy.git
   cd chproxy
   make build
   ```

2. **Run tests**
   ```bash
   make test
   ```

3. **Start development**
   ```bash
   make run
   ```

4. **Make changes**
   - Edit code
   - Run tests: `make test`
   - Run linters: `make lint`
   - Commit with clear message

5. **Submit PR**
   - Push to feature branch
   - Create PR with description
   - Address review feedback

---

*This guide is a living document. Update it as the project evolves!*
