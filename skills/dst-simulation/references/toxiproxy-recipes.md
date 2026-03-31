# Toxiproxy Fault Recipes

## Table of contents
- [Basic toxics](#basic-toxics)
- [Compound scenarios](#compound-scenarios)
- [Scripted fault sequences](#scripted-fault-sequences)

---

## Basic toxics

### Latency
Adds delay to all data through the proxy.
```bash
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"latency","type":"latency","stream":"downstream","attributes":{"latency":$MS,"jitter":$JITTER}}'
```
Use `stream: "upstream"` to delay requests, `"downstream"` to delay responses.

### Timeout
Stops all data and closes connection after timeout.
```bash
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"timeout","type":"timeout","attributes":{"timeout":$MS}}'
```

### Connection reset
Immediately resets the connection.
```bash
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"reset","type":"reset_peer","attributes":{"timeout":0}}'
```

### Bandwidth limit
Limits throughput in KB/s.
```bash
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"bandwidth","type":"bandwidth","attributes":{"rate":$KBPS}}'
```

### Slice (data corruption)
Slices data into small bits with optional delay.
```bash
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"slicer","type":"slicer","attributes":{"average_size":10,"size_variation":5,"delay":100}}'
```

### Remove a toxic
```bash
curl -s -X DELETE http://toxiproxy:8474/proxies/$PROXY/toxics/$TOXIC_NAME
```

### Remove all toxics from a proxy
```bash
# List and delete all
for toxic in $(curl -s http://toxiproxy:8474/proxies/$PROXY/toxics | python3 -c "import sys,json; [print(t['name']) for t in json.load(sys.stdin)]"); do
  curl -s -X DELETE http://toxiproxy:8474/proxies/$PROXY/toxics/$toxic
done
```

---

## Compound scenarios

### Degraded network (realistic)
Combine latency + bandwidth to simulate a degraded connection:
```bash
# 200ms latency + 50KB/s bandwidth
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"latency","type":"latency","attributes":{"latency":200,"jitter":50}}'
curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
  -d '{"name":"bandwidth","type":"bandwidth","attributes":{"rate":50}}'
```

### Intermittent failures
Alternate between working and broken states:
```bash
for i in $(seq 1 5); do
  # Break for 3 seconds
  curl -s -X POST http://toxiproxy:8474/proxies/$PROXY/toxics \
    -d '{"name":"reset","type":"reset_peer","attributes":{"timeout":0}}'
  sleep 3
  # Recover for 5 seconds
  curl -s -X DELETE http://toxiproxy:8474/proxies/$PROXY/toxics/reset
  sleep 5
done
```

### Cascading failure (DB slow → BE slow → FE timeout)
```bash
# First: slow the database
curl -s -X POST http://toxiproxy:8474/proxies/db-proxy/toxics \
  -d '{"name":"latency","type":"latency","attributes":{"latency":3000}}'
# Wait, then check: does backend propagate the slowness or circuit-break?
sleep 10
# Capture backend response times and FE behavior
```

---

## Scripted fault sequences

### Full resilience test (recommended default)
```bash
#!/bin/bash
set -euo pipefail
TOXI="http://toxiproxy:8474"
RESULTS=()

run_scenario() {
  local name=$1 proxy=$2 toxic_json=$3 pass_criteria=$4
  echo "=== Scenario: $name ==="

  # Inject fault
  curl -s -X POST "$TOXI/proxies/$proxy/toxics" -d "$toxic_json" > /dev/null

  # Run test traffic (customize per project)
  local status=$(curl -s -o /dev/null -w "%{http_code}" http://frontend:3000/api/data || echo "000")
  local time=$(curl -s -o /dev/null -w "%{time_total}" http://frontend:3000/api/data || echo "0")

  # Capture logs
  local errors=$(docker compose logs --since=10s 2>&1 | grep -ci "error\|exception\|panic\|fatal" || true)

  # Remove fault
  local toxic_name=$(echo "$toxic_json" | python3 -c "import sys,json; print(json.load(sys.stdin)['name'])")
  curl -s -X DELETE "$TOXI/proxies/$proxy/toxics/$toxic_name" > /dev/null

  # Wait for recovery
  sleep 3
  local recovery_status=$(curl -s -o /dev/null -w "%{http_code}" http://frontend:3000/api/data || echo "000")

  # Record result
  RESULTS+=("{\"name\":\"$name\",\"http_status\":$status,\"response_time\":$time,\"error_count\":$errors,\"recovery_status\":$recovery_status}")
  echo "  status=$status time=${time}s errors=$errors recovery=$recovery_status"
}

run_scenario "latency_fe_be" "backend-proxy" \
  '{"name":"lat","type":"latency","attributes":{"latency":2000}}' \
  "status != 500"

run_scenario "reset_fe_be" "backend-proxy" \
  '{"name":"rst","type":"reset_peer","attributes":{"timeout":0}}' \
  "graceful error"

run_scenario "db_timeout" "db-proxy" \
  '{"name":"tout","type":"timeout","attributes":{"timeout":5000}}' \
  "503 or retry"

run_scenario "external_api_down" "external-api-proxy" \
  '{"name":"rst","type":"reset_peer","attributes":{"timeout":0}}' \
  "fallback behavior"

run_scenario "bandwidth_limit" "backend-proxy" \
  '{"name":"bw","type":"bandwidth","attributes":{"rate":10}}' \
  "slow but functional"

echo "=== Results ==="
printf '[%s]\n' "$(IFS=,; echo "${RESULTS[*]}")"
```
