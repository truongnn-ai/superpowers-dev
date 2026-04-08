# Mutation Testing Setup

## Table of contents
- [Python (mutmut)](#python-mutmut)
- [JavaScript/TypeScript (Stryker)](#javascripttypescript-stryker)
- [Go (go-mutesting)](#go-go-mutesting)
- [Interpreting results](#interpreting-results)

---

## Python (mutmut)

### Install
```bash
pip install mutmut
```

### Run
```bash
# Basic run against src/ with tests in tests/
mutmut run \
  --paths-to-mutate=src/ \
  --tests-dir=tests/ \
  --runner="pytest -x -q" \
  --no-progress

# View results summary
mutmut results

# Export as JSON for parsing
mutmut results --json > mutation-results.json

# Show a specific surviving mutant
mutmut show <mutant-id>
```

### Parse survivors
```bash
mutmut results 2>&1 | grep "Survived" | while read line; do
  id=$(echo "$line" | grep -oP '\d+')
  mutmut show "$id" 2>&1
done
```

### Config (setup.cfg)
```ini
[mutmut]
paths_to_mutate=src/
tests_dir=tests/
runner=pytest -x -q
dict_synonyms=Struct,NamedStruct
```

---

## JavaScript/TypeScript (Stryker)

### Install
```bash
npm install --save-dev @stryker-mutator/core @stryker-mutator/vitest-runner
# or for Jest:
npm install --save-dev @stryker-mutator/core @stryker-mutator/jest-runner
```

### Config (stryker.config.mjs)
```javascript
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
export default {
  mutate: ['src/**/*.ts', '!src/**/*.test.ts', '!src/**/*.d.ts'],
  testRunner: 'vitest',
  reporters: ['json', 'clear-text', 'progress'],
  jsonReporter: { fileName: 'reports/mutation/mutation.json' },
  thresholds: { high: 70, low: 50, break: 40 },
  concurrency: 4,
  timeoutMS: 30000,
};
```

### Run
```bash
npx stryker run

# Results at reports/mutation/mutation.json
```

### Parse survivors from JSON
```bash
cat reports/mutation/mutation.json | python3 -c "
import sys, json
data = json.load(sys.stdin)
files = data.get('files', {})
for fname, fdata in files.items():
    for m in fdata.get('mutants', []):
        if m['status'] == 'Survived':
            loc = m['location']['start']
            print(f\"SURVIVED: {fname}:{loc['line']} - {m['mutatorName']}: {m.get('replacement', 'N/A')}\")
"
```

---

## Go (go-mutesting)

### Install
```bash
go install github.com/avito-tech/go-mutesting/cmd/go-mutesting@latest
```

### Run
```bash
go-mutesting ./...

# Or target specific packages
go-mutesting ./internal/pricing/...
```

### Alternative: gremlins
```bash
go install github.com/go-gremlins/gremlins/cmd/gremlins@latest
gremlins unleash --tags="" ./...
```

---

## Interpreting results

### Mutation score
```
score = killed_mutants / total_mutants
```

### What surviving mutants mean

| Mutation type | What it tests | If it survives, you're missing... |
|---|---|---|
| `>` → `>=` | Boundary conditions | Off-by-one test at threshold |
| `+` → `-` | Arithmetic correctness | Test with known expected output |
| `true` → `false` | Boolean logic | Test for both branches |
| `return x` → `return 0` | Return value significance | Assertion on return value |
| Remove function call | Side effect dependency | Test that side effect occurred |
| `==` → `!=` | Equality checks | Test for both equal and not-equal |

### Thresholds by code criticality

| Code area | Minimum score | Rationale |
|---|---|---|
| Payment, auth, data integrity | 60% | Bugs here cost money or trust |
| Core business logic | 40% | Logic errors affect users |
| CRUD, plumbing, config | 20% | Low risk, test the happy path |
| Generated boilerplate | 0% (skip) | Not worth mutating |

### Feeding survivors back to the agent

Format each survivor as:
```json
{
  "fix_category": "test_gap",
  "file": "src/pricing.py",
  "line": 67,
  "mutation": "Changed > to >=",
  "original_code": "if quantity > threshold:",
  "mutated_code": "if quantity >= threshold:",
  "suggested_action": "Add property test: what happens when quantity exactly equals threshold?"
}
```

The generating agent should either:
1. Add a test that kills the mutant, OR
2. Simplify the code to remove the untested branch (if the mutation is in dead/redundant code)
