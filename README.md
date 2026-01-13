# Interop Filter Utils

Testing utilities for the op-interop-filter service.

## Tools

### Spammer

Query generator that tests the filter by sending valid and invalid queries. It fetches real logs from the blockchain, then sends a configurable percentage of:
- **Valid queries**: Real logs with correct checksums (should be accepted)
- **Invalid queries**: Real logs with corrupted checksums (should be rejected)

The invalid percentage is configurable with optional noise to create more realistic patterns.

```bash
go run ./cmd/spammer --help
```

### Dashboard

Terminal-based monitoring UI for real-time observation of the filter and spammer metrics.

```bash
go run ./cmd/dashboard --help
```

## Running Integration Tests

This guide walks through running a manual integration test to verify the interop filter correctly accepts valid messages and rejects invalid ones.

### Prerequisites

- Go 1.22+
- RPC endpoints for the L2 chains you want to test (e.g., OP Sepolia, Unichain Sepolia)
- The op-interop-filter binary built from the optimism monorepo

### Step 1: Build the Interop Filter

```bash
cd /path/to/optimism/op-interop-filter
go build -v -o ./bin/op-interop-filter ./cmd
```

### Step 2: Run the Interop Filter

Start the filter with your L2 RPC endpoints. Use `--backfill-duration` to control how much history to sync (shorter = faster startup for testing).

```bash
./bin/op-interop-filter \
  --l2-rpcs "$OP_SEPOLIA_RPC" \
  --l2-rpcs "$UNI_SEPOLIA_RPC" \
  --rpc.port 7550 \
  --metrics.enabled \
  --metrics.port 7300 \
  --backfill-duration 3m \
  --log.level info
```

**Wait for backfill to complete.** Look for logs like:
```
lvl=info msg="Caught up, starting live ingestion" chain=11155420
lvl=info msg="Caught up, starting live ingestion" chain=1301
```

### Step 3: Run the Spammer

In a new terminal, start the spammer targeting one of your chains:

```bash
cd /path/to/interop-filter-utils

go run ./cmd/spammer \
  --l2-rpc "$OP_SEPOLIA_RPC" \
  --filter-rpc "http://localhost:7550" \
  --chain-id 11155420 \
  --num-queries 0 \
  --query-interval 1s \
  --safety-level cross-unsafe \
  --block-range 50 \
  --invalid-percent 30 \
  --noise-range 15 \
  --metrics.port 7301
```

**Key flags:**
- `--chain-id`: The chain ID to test (11155420 = OP Sepolia, 1301 = Unichain Sepolia)
- `--safety-level`: Use `cross-unsafe` to test cross-chain validation, or `unsafe` for local-only
- `--block-range`: How many recent blocks to sample from (smaller = more recent = less likely to hit timing issues)
- `--num-queries 0`: Run indefinitely (or set a number to stop after N queries)
- `--invalid-percent`: Target percentage of invalid queries (default: 50)
- `--noise-range`: Random variation around the target percentage (default: 10). E.g., `--invalid-percent 30 --noise-range 15` produces 15-45% invalid queries

### Step 4: Run the Dashboard (Optional)

In another terminal, start the dashboard to visualize metrics:

```bash
cd /path/to/interop-filter-utils

go run ./cmd/dashboard \
  --filter-metrics "http://localhost:7300/metrics" \
  --spammer-metrics "http://localhost:7301/metrics" \
  --refresh 2s
```

### Expected Results

The spammer logs should show progress like:
```
lvl=info msg=Progress queries=100 valid=62 invalid=38 errors=0
```

- **valid**: Number of valid queries (should all be accepted)
- **invalid**: Number of invalid queries (should all be rejected with "payload hash mismatch")
- **errors**: Unexpected errors (should be 0)

The ratio of valid/invalid depends on `--invalid-percent` and `--noise-range` flags.

**Success criteria:**
- Valid queries: ~100% accepted
- Invalid queries: ~100% rejected
- Errors: 0

### Troubleshooting

**"not yet cross-unsafe validated" errors:**

This happens when the spammer samples blocks with timestamps newer than the cross-unsafe timestamp. Solutions:
- Use `--safety-level unsafe` instead of `cross-unsafe`
- Use a smaller `--block-range` (e.g., 30) to sample more recent blocks
- Wait longer after backfill completes before starting the spammer
- Use a shorter `--backfill-duration` so both chains catch up closer together

**Timing with cross-unsafe:**

Cross-unsafe validation requires ALL chains to have caught up to a timestamp before messages at that timestamp are considered validated. If chains have different block times or backfill at different speeds, there can be a gap where recent blocks from one chain aren't yet cross-unsafe validated.

## Development

This module uses a local replace directive to depend on the optimism monorepo. Update the path in `go.mod` as needed:

```go
replace github.com/ethereum-optimism/optimism => ../optimism/develop
```

## Chain IDs

| Chain | Chain ID |
|-------|----------|
| OP Sepolia | 11155420 |
| Unichain Sepolia | 1301 |
| OP Mainnet | 10 |
| Base | 8453 |
| Base Sepolia | 84532 |
