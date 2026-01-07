# Interop Filter Utils

Testing utilities for the op-interop-filter service.

## Tools

### Dashboard

Terminal-based monitoring UI for real-time observation of the filter and spammer metrics.

```bash
go run ./cmd/dashboard --help
```

### Spammer

Query generator that tests the filter by sending valid and invalid queries.

```bash
go run ./cmd/spammer --help
```

## Development

This module uses a local replace directive to depend on the optimism monorepo. Update the path in `go.mod` as needed:

```go
replace github.com/ethereum-optimism/optimism => ../optimism/develop
```
