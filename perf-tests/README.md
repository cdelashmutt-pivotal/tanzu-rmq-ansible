# RabbitMQ Performance Testing

Performance test framework for validating RabbitMQ cluster reliability and replication under load.

Uses [rabbitmq-perf-test](https://github.com/rabbitmq/rabbitmq-perf-test) (AMQP queues) and [rabbitmq-stream-perf-test](https://github.com/rabbitmq/rabbitmq-stream-perf-test) (streams).

## Prerequisites

- Java 11+ on the machine running tests
- Network access to RabbitMQ nodes on ports 5672 (AMQP) and 5552 (streams)

## Setup

Install the perf-test tools:

```bash
ansible-playbook playbooks/install_perftest.yml
```

This downloads the JAR files to `perf-tests/tools/` and creates wrapper scripts.

## Running Tests

```bash
# List available scenarios
./perf-tests/run-test.sh

# Run a scenario against AZ-Cluster-1
./perf-tests/run-test.sh baseline --host 192.168.20.200

# Run against TX-Cluster-1
./perf-tests/run-test.sh baseline --host 192.168.20.206

# Add a label to distinguish runs
./perf-tests/run-test.sh baseline --host 192.168.20.200 --label "az-cluster-1-3ms"

# Test federation (publish to AZ-Cluster-1, consume from TX-Cluster-1)
./perf-tests/run-test.sh federation-test \
  --pub-host 192.168.20.200 \
  --con-host 192.168.20.206

# Pass extra args directly to perf-test
./perf-tests/run-test.sh baseline --host 192.168.20.200 -- --queue "my-test-queue"
```

Set `RMQ_PASSWORD` to avoid the password prompt:

```bash
export RMQ_PASSWORD="your-admin-password"
```

## Scenarios

| Scenario | Type | Description |
|----------|------|-------------|
| `baseline` | AMQP | 1 pub, 1 con, 1KB messages on quorum queue |
| `high-throughput` | AMQP | 5 pubs, 5 cons, small messages, max throughput |
| `large-messages` | AMQP | 64KB payloads, throttled rate |
| `latency-focus` | AMQP | Controlled 1k msg/s rate for accurate latency |
| `classic-queue` | AMQP | Non-replicated classic queue for comparison |
| `streams` | Stream | Stream with fan-out to 3 consumers |
| `federation-test` | AMQP | Cross-cluster federation replication |

## Comparing Results

```bash
# Show all results
./perf-tests/compare-results.sh

# Filter by scenario name
./perf-tests/compare-results.sh baseline

# Show last 5 results
./perf-tests/compare-results.sh --last 5
```

Results are saved to `perf-tests/results/` with timestamped filenames.

## Suggested Test Plan

### 1. Baseline Comparison

Establish performance baselines for each queue type:

```bash
./perf-tests/run-test.sh classic-queue --host 192.168.20.200 --label "az-cl-1"
./perf-tests/run-test.sh baseline --host 192.168.20.200 --label "az-cl-1-quorum"
./perf-tests/run-test.sh streams --host 192.168.20.200 --label "az-cl-1-stream"
```

### 2. Latency Impact

Compare Arizona (with metro latency) vs Texas clusters:

```bash
./perf-tests/run-test.sh baseline --host 192.168.20.200 --label "az-cl-1-3ms"
./perf-tests/run-test.sh baseline --host 192.168.20.206 --label "tx-cl-1-3ms"
```

### 3. Replication Overhead

Compare classic (non-replicated) vs quorum (replicated) on the same cluster:

```bash
./perf-tests/run-test.sh classic-queue --host 192.168.20.200 --label "no-replication"
./perf-tests/run-test.sh baseline --host 192.168.20.200 --label "quorum-replicated"
```

### 4. Federation Throughput

Measure cross-region federation replication performance:

```bash
./perf-tests/run-test.sh federation-test \
  --pub-host 192.168.20.200 \
  --con-host 192.168.20.206 \
  --label "az-to-tx"
```

### 5. Throughput Ceiling

Find the maximum throughput of each cluster:

```bash
./perf-tests/run-test.sh high-throughput --host 192.168.20.200 --label "az-cl-1-max"
./perf-tests/run-test.sh high-throughput --host 192.168.20.206 --label "tx-cl-1-max"
```

## Writing Custom Scenarios

Create a YAML file in `perf-tests/scenarios/`:

```yaml
name: my-test
description: "Description of what this tests"
type: amqp          # amqp or stream

duration: 60        # seconds
publishers: 1
pub_rate: 0         # 0 = unlimited
consumers: 1
consumer_rate: 0
message_size: 1000  # bytes
confirm: true
multi_ack_every: 100
queue_type: quorum  # classic, quorum, or stream
```

For stream tests, use `type: stream` and add stream-specific fields:

```yaml
type: stream
stream: my-stream-name
offset: first       # first, last, or next
```

## File Structure

```
perf-tests/
  run-test.sh              # Test runner script
  compare-results.sh       # Results comparison tool
  scenarios/               # Test scenario definitions
    baseline.yml
    high-throughput.yml
    large-messages.yml
    latency-focus.yml
    classic-queue.yml
    streams.yml
    federation-test.yml
  results/                 # Test output (git-ignored)
  tools/                   # Downloaded JARs (git-ignored)
```
