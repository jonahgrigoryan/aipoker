# Integration Checkpoints Reference

This document maps the testing checkpoints from `checkpoints.md` to the task flow in `tasks.md`, ensuring critical validation gates are executed at the right time.

---

## Overview

Each checkpoint represents a **STOP-AND-TEST** gate where you must validate functionality before proceeding. These gates prevent cascading failures and ensure you're building on a solid foundation.

---

## CHECKPOINT 1: After Task 2 (Configuration Manager)
**Location in tasks.md**: Task 2.5 → CHECKPOINT 1

### What to Validate
```bash
npm test -- config.test.ts
```

**Acceptance Criteria:**
- ✅ Load valid configuration without errors
- ✅ Hot-reload detects file changes within 1 second
- ✅ Invalid config triggers rollback (test with missing required fields)
- ✅ Schema validation catches type errors
- ✅ Subscription system notifies components on config changes

**Manual Verification:**
1. Start system with valid config → should load
2. Edit config file with invalid JSON → should rollback with clear error message
3. Change `strategy.alphaGTO` → verify Strategy Engine receives update notification

**Why This Matters:** Config drives all components. If config fails, everything downstream breaks.

**Do NOT proceed to Task 3 until this passes.**

---

## CHECKPOINT 2: After Task 3 (Vision System + Parser)
**Location in tasks.md**: Task 3.8 → CHECKPOINT 2

### What to Validate

```bash
# Run golden test suite
npm test -- vision.golden.test.ts

# Run parser tests
npm test -- parser.test.ts

# Run SafeAction tests
npm test -- safeaction.test.ts
```

**Acceptance Criteria:**
- ✅ **Card recognition: ≥99.5% accuracy** on golden image set (10k+ images)
- ✅ **Latency: ≤50ms @ P95** for frame capture + extraction
- ✅ **Pot/stack error: ≤1 BB @ P95**
- ✅ **Position assignment: ≤0.1% error rate** over 1000 test hands
- ✅ **Confidence gating:** Low-confidence frames trigger SafeAction
- ✅ **Occlusion detection:** >5% occlusion triggers SafeAction
- ✅ **SafeAction tests:** Check/fold logic correct for all game states

**Critical Metrics to Record:**
- P50/P95/P99 latency for each vision component
- Per-element confidence scores (cards, stacks, pot, positions)
- Occlusion detection accuracy

**Why This Matters:** Vision is the sensory input for the entire system. If vision is unreliable, every downstream decision will be wrong. This is the single most important checkpoint.

**Do NOT proceed to Task 4 until this passes.**

---

## CHECKPOINT 3: After Task 4 (GTO Solver)
**Location in tasks.md**: Task 4.7 → CHECKPOINT 3

### What to Validate

```bash
npm test -- solver.test.ts
```

**Acceptance Criteria:**
- ✅ **Cache hit rate: >80%** for preflop decisions in typical HU NLHE
- ✅ **Solve latency: ≤400ms @ P95** for cached + heuristic path
- ✅ **Output format:** GTOSolution contains valid action frequencies, EVs
- ✅ **Deep-stack logic:** Correctly adjusts when effective stack >100bb
- ✅ **Timeout handling:** Returns cached policy when budget exceeded

**Manual Verification:**
1. Query solver with common preflop situation (BTN vs BB, 100bb) → should hit cache
2. Query solver with postflop state → should return heuristic solution <400ms
3. Query with 200bb stack → verify different action abstractions vs 100bb

**Why This Matters:** GTO solver provides the mathematical foundation. If it's slow or incorrect, the bot will make poor decisions even with good LLM reasoning.

**Do NOT proceed to Task 5 until this passes.**

---

## CHECKPOINT 4: After Task 9 (Action Executor)
**Location in tasks.md**: Task 9.8 → CHECKPOINT 4

### What to Validate

```bash
# Test simulator executor
npm test -- executor.simulator.test.ts

# Test research UI executor (if enabled)
npm test -- executor.researchui.test.ts

# Integration test with custom simulator
npm run test:integration -- executor-simulator
```

**Acceptance Criteria:**
- ✅ **Simulator mode:** Successfully executes all action types (fold, check, call, raise)
- ✅ **Action verification:** Post-action state matches expected within ±1bb tolerance
- ✅ **Compliance checks:** Refuses to execute when window not in allowlist
- ✅ **Error recovery:** Correctly handles verification mismatches (retry once, then halt)
- ✅ **Bet sizing:** Quantizes continuous sizing to discrete set correctly

**Manual Verification (Simulator Mode):**
1. Start custom simulator from Task 0.7
2. Send action via executor API → verify hand progresses correctly
3. Check hand history logger receives execution results

**Manual Verification (Research UI Mode - Optional):**
1. Start allowed poker client (from allowlist)
2. Bot detects window and buttons correctly
3. Execute test action → verify action completes and verification passes

**Why This Matters:** Executor is the interface to the outside world. If it fails, the bot can't play. If verification fails, the bot will operate on incorrect state.

**Do NOT proceed to Task 10 until this passes.**

---

## CHECKPOINT 5: After Task 13 (Main Orchestration)
**Location in tasks.md**: Task 13.3 → CHECKPOINT 5

### What to Validate

```bash
# End-to-end integration test
npm run test:e2e

# Latency benchmark
npm run benchmark:decision-pipeline
```

**Acceptance Criteria:**
- ✅ **End-to-end latency: ≤2000ms @ P95** (Vision → Parser → GTO+Agents → Strategy → Executor)
- ✅ **Component communication:** All gRPC calls succeed, no dropped messages
- ✅ **Fallback policies:** When agents timeout, system falls back to GTO-only (α=1.0)
- ✅ **Panic stop:** Triggers correctly on 3 consecutive low-confidence frames
- ✅ **Compliance validation:** Startup checks refuse to run on prohibited sites

**Detailed Timing Breakdown:**
- Vision + Parser: ≤70ms @ P95
- GTO Solver: ≤400ms @ P95
- Agent Coordinator: ≤1200ms @ P95 (agents run in parallel)
- Strategy Engine: ≤100ms @ P95
- Action Executor: ≤30ms @ P95
- Buffer: 200ms

**Manual Verification:**
1. Start full system in simulator mode
2. Play 100 hands end-to-end
3. Verify no crashes, all hands complete
4. Check logs show correct timing distributions (P50/P95/P99)
5. Trigger panic stop by injecting low-confidence frames → verify system halts correctly

**Why This Matters:** This validates the entire pipeline works together. Individual components may work, but integration issues can still break the system.

**Do NOT proceed to Task 15 (Evaluation) until this passes.**

---

## Checkpoint Summary Table

| Checkpoint | After Task | Component(s) | Critical Metrics | Gate Criteria |
|------------|------------|--------------|------------------|---------------|
| 1 | Task 2 | Config Manager | Load/reload/rollback | All config tests pass |
| 2 | Task 3 | Vision + Parser | 99.5% accuracy, <50ms latency | Vision golden tests pass |
| 3 | Task 4 | GTO Solver | >80% cache hit, <400ms solve | Solver tests pass |
| 4 | Task 9 | Action Executor | Executes all actions, verification works | Executor integration tests pass |
| 5 | Task 13 | Full Pipeline | <2000ms end-to-end @ P95 | 100-hand simulation completes |

---

## When to Skip a Checkpoint

**Never.** These checkpoints are mandatory. Skipping them will result in:
- Wasted time debugging mysterious failures later
- Cascading bugs that are hard to isolate
- Risk of building an entire system that doesn't work

If a checkpoint fails:
1. **Stop** immediately
2. **Debug** the failing component in isolation
3. **Fix** the issue
4. **Re-test** until it passes
5. **Only then** proceed to the next task

---

## Integration Tests vs Unit Tests

- **Unit tests:** Test individual functions/classes in isolation
- **Integration tests:** Test multiple components working together
- **Checkpoints:** Validate critical integration points before proceeding

Example:
- Unit test: `testCardRecognition()` verifies card classifier works on single image
- Integration test: `testVisionPipeline()` verifies capture → extract → parse flow
- Checkpoint 2: Validates vision system meets all accuracy/latency requirements on full golden dataset

---

## Adding New Checkpoints

As the project evolves, you may want to add additional checkpoints:

1. After Task 5 (Agents): Validate agent coordinator correctly aggregates LLM outputs
2. After Task 8 (Strategy): Validate blending algorithm produces correct distributions
3. After Task 14 (Monitoring): Validate metrics collection and alerting work

To add a checkpoint:
1. Define acceptance criteria
2. Add `CHECKPOINT X` marker in tasks.md after relevant task
3. Document it in this file
4. Create corresponding test suite
5. Add to CI pipeline to enforce gate

---

## CI Pipeline Integration

The CI pipeline should enforce these checkpoints:

```yaml
# .github/workflows/ci.yml (example)
jobs:
  checkpoint-1:
    name: Checkpoint 1 - Config Manager
    steps:
      - run: npm test -- config.test.ts
  
  checkpoint-2:
    name: Checkpoint 2 - Vision System
    needs: checkpoint-1
    steps:
      - run: npm test -- vision.golden.test.ts
      - run: npm test -- parser.test.ts
      - run: npm test -- safeaction.test.ts
      - name: Verify latency targets
        run: |
          npm run benchmark:vision
          # Fail if P95 > 50ms
  
  checkpoint-3:
    name: Checkpoint 3 - GTO Solver
    needs: checkpoint-2
    steps:
      - run: npm test -- solver.test.ts
  
  checkpoint-4:
    name: Checkpoint 4 - Action Executor
    needs: checkpoint-3
    steps:
      - run: npm test -- executor.simulator.test.ts
      - run: npm run test:integration -- executor-simulator
  
  checkpoint-5:
    name: Checkpoint 5 - Full Pipeline
    needs: checkpoint-4
    steps:
      - run: npm run test:e2e
      - run: npm run benchmark:decision-pipeline
      - name: Verify end-to-end latency
        run: |
          # Fail if P95 > 2000ms
```

This ensures no code can be merged that breaks a checkpoint.
