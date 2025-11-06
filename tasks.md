# Implementation Plan

## Overview

This implementation plan follows a phased approach to manage complexity and accelerate time-to-MVP:

**Phase 1 (MVP)**: Precomputed GTO ranges + heuristics, 3 LLM agents, Windows-only research UI, custom simulator
**Phase 2 (Optional)**: Full CFR solver, cross-platform support, advanced opponent modeling, 10M hand evaluation

Task 0 establishes concrete technology decisions to eliminate ambiguity and enable immediate implementation.

---

- [ ] 0. Technology Stack and Architectural Decisions
  - [ ] 0.1 Select runtime environment and primary language
    - Decision: Node.js (v20 LTS) for main orchestrator, TypeScript for type safety
    - Decision: Python 3.11 for vision and GTO solver microservices
    - Rationale: Node.js handles async I/O well for LLM calls; Python ecosystem for vision/ML
    - _Requirements: 0.1, 10.2_
  
  - [ ] 0.2 Select vision technology stack
    - Decision: Python OpenCV 4.8+ for frame capture and preprocessing
    - Decision: ONNX Runtime with pretrained CNN models for card/digit recognition
    - Decision: Tesseract OCR 5.x as fallback for unsupported themes
    - Decision: MSS (mss library) for cross-platform screen capture
    - Model: Train/acquire ONNX models for 52-card classification (rank+suit) and digit recognition (0-9, K, M, B suffixes)
    - Rationale: OpenCV is industry standard, ONNX enables model portability, Tesseract provides fallback
    - _Requirements: 1.1, 1.3, 1.5_
  
  - [ ] 0.3 Select GTO solver approach and implementation strategy
    - Decision Phase 1: Use precomputed range charts for preflop (export from PioSOLVER/GTO+ or use published ranges)
    - Decision Phase 1: Implement rule-based postflop heuristics (c-bet frequency, equity-based decisions, pot odds)
    - Decision Phase 2 (optional): Integrate open-source CFR library (OpenSpiel, PokerKit) for subgame solving
    - Rationale: Precomputed + heuristics gets MVP running fast; defer full CFR to phase 2 after validation
    - Budget: Allow 400ms for cache lookup + heuristic calculation initially
    - _Requirements: 2.1, 2.2, 2.6_
  
  - [ ] 0.4 Select LLM providers and models
    - Decision: OpenAI GPT-4o-mini as primary agent (cost-effective, fast)
    - Decision: Anthropic Claude 3.5 Haiku as secondary agent (diversity)
    - Decision: OpenAI GPT-4o as third agent (higher reasoning, used selectively)
    - Cost cap: $0.10 per hand maximum (agent prompts ~500 tokens input, ~100 tokens output each)
    - Rationale: Mix of speed and capability; Haiku adds architectural diversity; cost-controlled
    - _Requirements: 3.1, 3.2_
  
  - [ ] 0.5 Select initial platform support scope
    - Decision: Target Windows 10/11 only for initial release
    - Decision: Use @nut-tree/nut-js for research UI automation (cross-platform foundation for future)
    - Decision: Linux and macOS support deferred to phase 2
    - Rationale: Focus on single platform to accelerate development; nut-js keeps future expansion feasible
    - _Requirements: 5.2, 5.7.1, 5.7.6_
  
  - [ ] 0.6 Select or build evaluation simulator
    - Decision: Build minimal custom Python simulator using PokerKit library for game rules
    - Features: HU NLHE support, deterministic RNG, hand history export, API interface
    - Opponent types: Fixed-strategy bots (tight-aggressive 20/16, loose-passive 45/10, calling station 60/5)
    - Rationale: Full control over test environment, reproducible results, no external dependencies
    - _Requirements: 9.1, 9.2, 9.6_

- [ ] 1. Set up project structure and core interfaces
  - Create monorepo structure: /orchestrator (Node.js/TS), /vision-service (Python), /solver-service (Python), /shared
  - Define TypeScript interfaces for core types: Card, Action, Position, Street, GameState, RNG
  - Implement configuration schema and validation using JSON Schema (ajv library)
  - Set up build system: pnpm workspaces, esbuild for TS, Poetry for Python, Docker multi-stage builds
  - Set up gRPC interfaces between Node.js orchestrator and Python microservices (vision, solver)
  - Pin all dependencies: package.json (exact versions), poetry.lock, Dockerfile base images with SHA256
  - _Requirements: 0.1, 10.2_

- [ ] 2. Implement Configuration Manager
  - [ ] 2.1 Create configuration loading and validation
    - Implement BotConfig interface with all required sections
    - Write JSON Schema validator for configuration structure
    - Implement file loading with error handling
    - _Requirements: 8.1, 8.6_
  
  - [ ] 2.2 Implement hot-reload with rollback
    - Add file watcher for configuration changes
    - Implement validation-before-apply logic
    - Create rollback mechanism to restore last-known-good config
    - _Requirements: 8.2, 8.6_
  
  - [ ] 2.3 Add configuration subscription system
    - Implement pub-sub pattern for config change notifications
    - Create getter methods with key path support
    - _Requirements: 8.2_
  
  - [ ] 2.4 Write unit tests for Config schema validator
    - Test validation with valid and invalid configurations
    - Test rollback on validation failure
    - Test hot-reload behavior
    - _Requirements: 8.6_

- [ ] 3. Implement Vision System and Game State Parser with model preloading
  - [ ] 3.1 Create layout pack system
    - Define LayoutPack JSON schema with ROI definitions and version field
    - Implement layout pack loader with version validation
    - Add DPI and theme calibration routine using reference anchor detection
    - Create sample layout packs for custom simulator (from task 0.6)
    - _Requirements: 1.5_
  
  - [ ] 3.2 Implement frame capture and element extraction
    - Integrate MSS library for screen capture (target 60fps capability)
    - Implement ROI-based extraction using OpenCV cv2.imread and slicing
    - Integrate ONNX Runtime with card classification model (52-class output) and digit recognition model (15-class: 0-9, K, M, B, comma, decimal)
    - Preload ONNX models and warm inference sessions on startup to avoid first-hand latency spikes
    - Implement Tesseract OCR fallback for stack/pot values when CNN confidence <0.995
    - Build or source ONNX models: Card classifier (~5MB, MobileNetV3), Digit classifier (~2MB, CRNN)
    - _Requirements: 1.1, 1.3_
  
  - [ ] 3.3 Add confidence scoring and occlusion detection
    - Implement per-element confidence calculation based on match quality
    - Add occlusion detection by analyzing ROI pixel variance
    - Create VisionOutput interface with confidence and latency tracking
    - _Requirements: 1.2, 1.6, 1.7, 1.9_
  
  - [ ] 3.4 Implement Game State Parser
    - Convert VisionOutput to structured GameState JSON
    - Implement position assignment logic (BTN/SB/BB inference)
    - Add state-sync error tracking across consecutive frames
    - Compute legal actions based on game rules
    - _Requirements: 1.4, 1.8_
  
  - [ ] 3.5 Add confidence gating and SafeAction trigger
    - Implement shouldTriggerSafeAction logic (confidence <0.995 or occlusion >5%)
    - Create SafeAction policy: preflop check-or-fold, postflop check-or-fold
    - Honor forced actions (blinds auto-posted, all-in only when committed)
    - _Requirements: 1.2, 10.3, 10.6_
  
  - [ ] 3.7 Write unit tests for SafeAction policy
    - Test SafeAction selection for various game states
    - Test forced action handling (blinds, all-in commitments)
    - Test confidence gating triggers
    - _Requirements: 10.6_
  
  - [ ] 3.6 Create vision golden test suite
    - Build fixed image set covering various game states
    - Write tests for per-element confidence scoring
    - Test occlusion gating and state-sync checks
    - _Requirements: 1.2, 1.6, 1.7, 1.8, 1.9_

- [ ] 4. Implement GTO Solver (Phase 1: Precomputed + Heuristics)
  - [ ] 4.1 Create cache system for precomputed solutions
    - Define state fingerprinting algorithm with fields: street, positions, stacks bucketed, pot, blinds, board cards, action history discretized, SPR, version byte
    - Implement cache storage format (compressed JSON or msgpack for phase 1)
    - Create cache loader and query interface with fuzzy matching for nearest stack/pot bucket
    - Build preflop range cache: Import GTO ranges from published sources (Upswing, GTOWizard free charts) or export from PioSOLVER trial
    - Preload preflop cache on startup to avoid first-hand latency spikes
    - _Requirements: 2.1, 2.6_
  
  - [ ] 4.2 Implement postflop heuristic solver (Phase 1)
    - Implement equity calculation using PokerKit equity evaluator
    - Create rule-based decision trees: c-bet frequency by board texture, pot odds for draws, value-bet-or-check equity thresholds
    - Add position-aware adjustments (IP vs OOP aggression factors)
    - Return GTOSolution-compatible output with heuristic-derived action frequencies
    - Target: <400ms for equity calc + heuristic evaluation
    - _Requirements: 2.2_
  
  - [ ] 4.2b Integrate CFR subgame solver (Phase 2 - optional)
    - Integrate OpenSpiel CFR solver or PokerKit solver
    - Add game tree abstraction (card bucketing via k-means on equity distributions, action abstraction to 3-4 bet sizes)
    - Implement budget-aware solving with early stopping when 400ms budget exhausted
    - Cache solved subgames for reuse within session
    - _Requirements: 2.2_
  
  - [ ] 4.3 Add deep-stack adjustments
    - Implement effective stack calculation
    - Create deep-stack action abstractions (more bet sizes)
    - Apply adjustments when effective stack >100bb
    - _Requirements: 2.4_
  
  - [ ] 4.4 Create GTOSolution output interface
    - Return action frequencies, EVs, and regret deltas
    - Track computation time and source (cache vs subgame)
    - Return cached policy when budget would be exceeded
    - _Requirements: 2.5, 2.6_

- [ ] 5. Implement Agent Coordinator and LLM integration
  - [ ] 5.1 Create LLM agent interface and prompt system
    - Define AgentOutput interface with reasoning, recommendation, sizing, confidence
    - Implement prompt template system with persona injection and game state formatting
    - Create three personas mapped to models from task 0.4:
      - "GTO Purist" → GPT-4o-mini: Emphasizes equilibrium play, range balance, unexploitability
      - "Exploitative Aggressor" → Claude 3.5 Haiku: Looks for opponent weaknesses, suggests deviations
      - "Risk Manager" → GPT-4o (selective use): Conservative EV calculations, bankroll considerations
    - Include opponent statistics in prompts when available (VPIP, PFR, 3bet, fold-to-cbet)
    - _Requirements: 3.1_
  
  - [ ] 5.2 Implement parallel agent querying
    - Set up concurrent API calls using Promise.all() to OpenAI and Anthropic SDKs
    - Add timeout handling (3s per agent using AbortController)
    - Implement exponential backoff retry logic for 429/500 errors (max 1 retry to preserve time budget)
    - Add API key rotation support for rate limit distribution
    - _Requirements: 3.2_
  
  - [ ] 5.3 Add JSON schema validation
    - Define strict output schema for agent responses using JSON Schema
    - Implement schema validator using ajv library with strict mode
    - Discard malformed outputs and count toward timeout (log warning with agent name and raw output)
    - Gracefully handle partial outputs when agent reasoning is valid but recommendation malformed
    - _Requirements: 3.7_
  
  - [ ] 5.4 Implement agent weighting and aggregation
    - Create weighted voting system: initial weights [0.33, 0.33, 0.34], updated per Brier scores
    - Implement Brier score tracking per agent across all decisions (store in session metrics)
    - Add calibration method using labeled validation set (minimum 1000 hands for statistical significance)
    - Generate AggregatedAgentOutput with consensus metric (0-1 based on action distribution entropy)
    - _Requirements: 3.4, 3.5_
  
  - [ ] 5.5 Add cost controls and circuit breaker for LLM agents
    - Implement per-decision token counting: estimate input tokens (state JSON ~400-600), cap output at 150 tokens
    - Track cumulative cost per session: ($0.15/1M input + $0.60/1M output) for GPT-4o-mini, ($0.25/1M + $1.25/1M) for Haiku, ($2.50/1M + $10/1M) for GPT-4o
    - Drop GPT-4o agent if per-hand cost exceeds $0.10 or session cost exceeds $50
    - Add circuit breaker: trip after 5 consecutive timeouts/errors per agent, force α=1.0 (GTO-only) if all agents tripped
    - Alert operator when circuit breaker trips
    - _Requirements: 4.6_
  
  - [ ] 5.6 Write unit tests for agent JSON schema validator
    - Test schema validation with valid and malformed JSON
    - Test timeout counting for malformed outputs
    - Test circuit breaker triggering
    - _Requirements: 3.7_

- [ ] 6. Implement Time Budget Tracker
  - [ ] 6.1 Create budget allocation and tracking
    - Define BudgetAllocation interface with per-component budgets
    - Implement high-resolution timer (performance.now())
    - Create budget tracking with elapsed/remaining methods
    - _Requirements: 4.6_
  
  - [ ] 6.2 Add preemption logic
    - Implement shouldPreempt() checks for each component
    - Add dynamic budget adjustment when perception overruns
    - Track actual durations for P50/P95/P99 analysis
    - _Requirements: 4.1, 4.6_
  
  - [ ] 6.3 Write unit tests for Time Budget Tracker
    - Test budget allocation and tracking
    - Test preemption logic under various timing scenarios
    - Test dynamic budget adjustment
    - _Requirements: 4.6_

- [ ] 7. Implement Risk Guard
  - [ ] 7.1 Create risk limit enforcement
    - Implement RiskGuard class with bankroll and session tracking
    - Add checkLimits() method called before action finalization
    - Trigger panic stop when limits exceeded
    - _Requirements: 10.4_
  
  - [ ] 7.2 Write unit tests for RiskGuard
    - Test bankroll tracking and limit enforcement
    - Test session limit enforcement
    - Test panic stop triggering
    - _Requirements: 10.4_

- [ ] 8. Implement Strategy Engine
  - [ ] 8.1 Create blending algorithm
    - Implement blend() method: α × GTO + (1-α) × Exploit
    - Add runtime α adjustment within bounds [0.3, 0.9]
    - _Requirements: 4.2_
  
  - [ ] 8.2 Implement action selection and bet sizing
    - Create selectAction() using seeded RNG to sample from distribution
    - Implement quantizeBetSize() to map continuous sizing to discrete set
    - Support per-street bet sizing sets from config
    - Enforce bet size legality: clamp to site min increment and table caps
    - Assert bet size validity before passing to executor
    - _Requirements: 4.7_
  
  - [ ] 8.3 Add divergence detection and logging
    - Implement computeDivergence() using total variation distance
    - Log full trace when divergence >30pp (state, seeds, model hashes)
    - _Requirements: 4.3_
  
  - [ ] 8.4 Integrate risk checks and fallbacks
    - Call RiskGuard.checkLimits() before finalizing decision
    - Implement GTO-only fallback when agents timeout (α=1.0)
    - Return SafeAction when risk limits exceeded
    - _Requirements: 4.5, 10.3_
  
  - [ ] 8.6 Add opponent modeling statistics store (optional)
    - Create per-villain frequency tracking (VPIP, PFR, 3bet, etc.)
    - Store node-level action weights
    - Feed statistics to Strategy Engine and agent prompts for exploitation
    - _Requirements: 4.2_
  
  - [ ] 8.5 Create StrategyDecision output
    - Package final action with reasoning breakdown
    - Include timing for GTO, agents, and synthesis
    - Ensure end-to-end decision within 2s at P95
    - _Requirements: 4.1_

- [ ] 9. Implement Action Executor
  - [ ] 9.1 Create simulator/API executor
    - Implement executeSimulator() for direct API calls to custom Python simulator (from task 0.6)
    - Implement executeAPI() for REST/WebSocket interfaces to external platforms with API support
    - Add action translation logic (StrategyDecision → simulator API commands)
    - Define API interface contract: POST /action with {handId, action, amount?}
    - _Requirements: 5.1_
  
  - [ ] 9.2 Add research UI mode with compliance checks (Windows only, Phase 1)
    - Gate research UI behind build flag --research-ui (default off in package.json scripts)
    - Implement executeResearchUI() using @nut-tree/nut-js for mouse/keyboard automation
    - Add environment validation: check active window title against config.execution.researchUIAllowlist
    - Enforce compliance: refuse execution if window title not in allowlist, log ComplianceError with detected window
    - Add startup banner when research UI mode enabled: "RESEARCH UI MODE - Only use on permitted platforms"
    - _Requirements: 5.2, 5.3, 0.2, 0.3, 0.4_
  
  - [ ] 9.3 Implement action verification
    - Capture post-action frame using vision service (wait 500ms for UI update)
    - Parse state and compare to expected state: pot size changed by bet amount, stack reduced, action history updated
    - Define strict equality rules: pot ±1bb tolerance, stack ±1bb tolerance, action type exact match
    - Re-evaluate once on mismatch with bounded retry (wait additional 500ms, re-capture), then halt with VerificationError
    - _Requirements: 5.4, 5.5_
  
  - [ ] 9.4 Add bet sizing precision
    - Support fold, check, call actions via button clicks
    - Implement raise/bet with exact amounts: click raise button, clear input field, type amount, confirm
    - Implement bet sizing from discrete set (map continuous recommendation to nearest allowed size)
    - Handle all-in as special case: detect all-in button or max bet size
    - _Requirements: 5.6_

  - [ ] 9.5 Implement Window Manager (Windows Phase 1)
    - Create window detection using Windows API via edge-js or node-ffi-napi (EnumWindows, GetWindowText)
    - Alternative: Use @nut-tree/nut-js screen.findOnScreen() with window title patterns
    - Add window validation: match process name (process.name) and title (regex patterns) from layout pack
    - Implement coordinate conversion from ROI (relative to window) to screen space (window.position + ROI offset)
    - Add window.focus() call before any click to ensure foreground status
    - _Requirements: 5.7.1, 5.7.2, 5.7.5_

  - [ ] 9.6 Extend Vision System for action buttons
    - Add template matching for action buttons using OpenCV matchTemplate with templates from layout pack
    - Implement turn state detection: look for timer animation, highlighted player border, or "Your turn" text via OCR
    - Update VisionOutput interface with actionButtons map and turnState object
    - Define ButtonInfo with screenCoords, isEnabled (color-based detection), isVisible, confidence, text (OCR of button label)
    - _Requirements: 5.7.3, 5.7.4_

  - [ ] 9.7 Enhance research UI executor
    - Implement window focus management using nut-js window.focus() or Windows SetForegroundWindow
    - Add turn waiting logic: poll vision.detectTurnState() every 200ms, timeout after 30s (fold equity lost)
    - Create cross-platform mouse/keyboard automation using nut-js mouse.click(), keyboard.type()
    - Handle bet sizing input fields: triple-click to select all, type amount, press Enter or click confirm button
    - Add randomized human-like delays: 800ms-1500ms between decision and click, 50ms-200ms between keystrokes
    - _Requirements: 5.7.6, 5.7.7_

- [ ] 10. Implement Hand History Logger
  - [ ] 10.1 Create logging data structures
    - Define HandRecord interface with all required fields
    - Include raw state, parsed state, solver output, agent texts, decision, timings
    - Add metadata: RNG seeds, model hashes, config snapshot
    - _Requirements: 6.1, 6.3_
  
  - [ ] 10.2 Implement log persistence
    - Create append-only log files (one per session)
    - Persist hand records within 1s of hand completion
    - _Requirements: 6.2_
  
  - [ ] 10.3 Add PII redaction
    - Implement redactPII() to remove player names, IDs, IP addresses
    - Replace with position labels
    - _Requirements: 6.5_
  
  - [ ] 10.4 Implement export formats
    - Create JSON exporter (pretty-printed)
    - Create ACPC format exporter
    - _Requirements: 6.4_
  
  - [ ] 10.5 Add metrics tracking
    - Track win rate (bb/100), EV accuracy, decision quality
    - Compute P50/P95/P99 latency distributions per module
    - Generate SessionMetrics summaries
    - _Requirements: 6.7, 6.8_
  
  - [ ] 10.6 Implement retention policy
    - Add configurable retention window
    - Auto-delete logs older than retention period
    - _Requirements: 6.6_

- [ ] 11. Implement Health Monitor
  - [ ] 11.1 Create health check system
    - Implement checkComponent() for each major component
    - Add periodic health checks every 5-10 seconds
    - Return HealthStatus with status and message
    - _Requirements: 7.3_
  
  - [ ] 11.2 Implement safe mode
    - Create enterSafeMode() to lock Action Executor
    - Continue logging in safe mode
    - Add exitSafeMode() for manual recovery
    - _Requirements: 7.2_
  
  - [ ] 11.3 Add panic stop logic
    - Trigger on 3 consecutive frames with confidence <0.99
    - Trigger on bankroll/session limit exceeded
    - Halt all automated play and require manual restart
    - _Requirements: 10.5_
  
  - [ ] 11.4 Create status dashboard (optional)
    - Build web UI showing real-time component status
    - Display recent hands and metrics
    - Show alerts for degraded/failed components
    - _Requirements: 7.5_

- [ ] 12. Implement deterministic replay and RNG seeding
  - [ ] 12.1 Create seeded RNG system
    - Implement RNG interface with seed support
    - Generate seed from handId + sessionId hash
    - Use seeded RNG for all randomness (action selection, timing)
    - _Requirements: 10.1_
  
  - [ ] 12.2 Add model versioning and hashing
    - Track LLM model weights hashes
    - Track vision model versions
    - Track GTO cache versions
    - Include in HandRecord metadata
    - _Requirements: 10.2_

- [ ] 13. Wire components into main decision pipeline
  - [ ] 13.1 Create main orchestration loop
    - Implement pipeline: Vision → Parser → (GTO + Agents) → Strategy → Executor → Logger
    - Integrate Time Budget Tracker across all components
    - Add error handling and fallback policies
    - _Requirements: 7.1, 7.4_
  
  - [ ] 13.2 Add compliance validation at startup
    - Validate environment against config allowlist
    - Check for prohibited sites
    - Halt if compliance check fails
    - _Requirements: 0.5_
  
  - [ ] 13.3 Implement end-to-end decision flow
    - Ensure 2s deadline met at P95
    - Verify all components communicate correctly
    - Test with sample game states
    - _Requirements: 4.1_

- [ ] 14. Implement monitoring and observability
  - [ ] 14.1 Add structured logging
    - Implement log levels (DEBUG, INFO, WARN, ERROR, CRITICAL)
    - Log per-hand decisions and metrics
    - Log component failures and fallbacks
    - _Requirements: 7.1_
  
  - [ ] 14.2 Create metrics collection
    - Track performance metrics (latency P50/P95/P99, throughput)
    - Track decision quality metrics (win rate, EV accuracy, exploitability)
    - Track system health metrics (uptime, safe mode triggers, panic stops)
    - Track cost metrics (LLM tokens, solver time, $/1k hands)
    - _Requirements: 6.8_
  
  - [ ] 14.3 Add alerting system (optional)
    - Implement Slack/email alerts for critical errors
    - Add dashboard warnings for degraded performance
    - Generate daily summary reports
    - _Requirements: 7.5_

- [ ] 15. Create evaluation framework
  - [ ] 15.1 Implement offline evaluation smoke test
    - Use custom Python simulator from task 0.6 with PokerKit game engine
    - Implement three fixed-strategy opponents:
      - Tight-Aggressive (20% VPIP, 16% PFR): folds weak hands, aggressive with strong hands
      - Loose-Passive (45% VPIP, 10% PFR): calls frequently, rarely raises
      - Calling Station (60% VPIP, 5% PFR): calls almost everything, minimal aggression
    - Run 10,000 hand smoke test (3,333 hands vs each opponent, ~2-4 hours runtime)
    - Track win rate with 95% confidence intervals using t-distribution
    - Wire evaluation targets: ≥3bb/100 vs static pool, ≥0bb/100 vs mixed-GTO, ε≤0.02
    - Generate report: win rate per opponent, aggregate bb/100, latency P50/P95/P99, cost per hand
    - _Requirements: 9.1, 9.2, 9.6, 9.7, 9.8_
  
  - [ ] 15.1b Expand offline evaluation to full 10M hand suite (optional)
    - Add GTO baseline opponent using preflop ranges + heuristics (mirror of bot's Phase 1 solver)
    - Add mixed-GTO opponent (80% GTO, 20% random deviations)
    - Run 10M hand simulations (2M hands per opponent, ~200-300 hours runtime)
    - Measure exploitability: play bot against itself, verify win rate ≈0 bb/100
    - Compare against baseline CFR bot: import published CFR strategy, measure win rate
    - Generate academic-style report with tables, confidence intervals, and strategy analysis
    - _Requirements: 9.1, 9.2, 9.6, 9.7, 9.8_
  
  - [ ] 15.2 Implement shadow mode harness
    - Create dataset loader for hand histories in ACPC format
    - Build harness: replay hand states, invoke bot decision pipeline without execution, compare to actual actions
    - Support private/sim datasets only (no scraped hands from prohibited sites - verify source provenance)
    - Implement decision comparison metrics: exact match rate, EV difference (estimated via equity calculations)
    - Target: 1000 hands minimum for validation, 100k hands for comprehensive evaluation
    - _Requirements: 9.3_
  
  - [ ] 15.2b Run full shadow mode evaluation (optional)
    - Run for 100k hands minimum
    - Compare bot decisions to baseline decisions
    - Measure agreement rate and EV difference
    - _Requirements: 9.3_
  
  - [ ] 15.3 Create A/B testing framework (optional)
    - Support configuration variants (GTO-only vs blend, subgame vs no-subgame)
    - Run parallel experiments
    - Report results with confidence intervals
    - _Requirements: 9.4, 9.5_

- [ ] 16. Create deployment artifacts and CI pipeline
  - [ ] 16.1 Create Docker containers
    - Write Dockerfiles for each component
    - Pin all dependencies and base images
    - Include build metadata (timestamp, git commit)
    - _Requirements: 10.2_
  
  - [ ] 16.2 Set up container orchestration
    - Create Docker Compose configuration
    - Define inter-component communication (gRPC)
    - Configure volume mounts for cache, logs, config
    - _Requirements: 10.2_
  
  - [ ] 16.3 Add environment variable management
    - Set up .env file for API keys
    - Document required environment variables
    - Implement key rotation support
    - _Requirements: 10.2_
  
  - [ ] 16.4 Create CI pipeline
    - Set up build job for all components
    - Run vision golden tests in CI
    - Run required unit tests (SafeAction, RiskGuard, BudgetTracker, Config validator, Agent schema validator)
    - Fail build if latency P95 exceeds budget allocations
    - _Requirements: 1.2, 4.6, 10.4, 10.6, 8.6, 3.7_

- [ ] 17. Create documentation and examples (optional)
  - [ ] 17.1 Write configuration guide (optional)
    - Document all config parameters
    - Provide example configs for different game types
    - Explain tuning strategies (α adjustment, bet sizing, agent personas)
    - _Requirements: 8.1, 8.3, 8.4_
  
  - [ ] 17.2 Create operator manual (optional)
    - Document startup procedures
    - Explain monitoring dashboard
    - Provide troubleshooting guide
    - Document safe mode and panic stop recovery
    - _Requirements: 7.2, 7.5_
  
  - [ ] 17.3 Write developer guide (optional)
    - Document architecture and component interfaces
    - Explain how to add new LLM agents
    - Explain how to create new layout packs
    - Document evaluation procedures
    - _Requirements: 9.1, 9.2, 9.3_

---

## Implementation Milestones

### Milestone 1: Foundation (Tasks 0-2)
**Goal**: Establish technology stack and configuration infrastructure  
**Completion criteria**: Config manager loads/validates/hot-reloads configuration, all dependencies pinned  
**Estimated effort**: 1-2 weeks

### Milestone 2: Perception Pipeline (Task 3)
**Goal**: Vision system extracts game state with required accuracy and latency  
**Completion criteria**: 99.5% card recognition accuracy, <50ms extraction latency @ P95, SafeAction policy tested  
**Estimated effort**: 3-4 weeks

### Milestone 3: Decision Core (Tasks 4-8)
**Goal**: GTO solver + LLM agents + strategy blending produce decisions within time budget  
**Completion criteria**: Preflop ranges cached, postflop heuristics implemented, 3 LLM agents integrated, 2s end-to-end @ P95  
**Estimated effort**: 4-6 weeks

### Milestone 4: Execution & Logging (Tasks 9-10)
**Goal**: Execute actions via simulator/UI and persist comprehensive logs  
**Completion criteria**: Simulator API executor working, research UI executor tested on custom simulator, hand history logger persisting all fields  
**Estimated effort**: 3-4 weeks

### Milestone 5: Reliability & Observability (Tasks 11-14)
**Goal**: System handles errors gracefully and provides operational visibility  
**Completion criteria**: Health monitor detects failures, safe mode locks executor, metrics tracked per module, panic stop tested  
**Estimated effort**: 2-3 weeks

### Milestone 6: Evaluation & MVP (Tasks 15-16)
**Goal**: Validate system performance against targets and create deployable artifacts  
**Completion criteria**: 10k hand smoke test passes (≥3bb/100 vs static pool), Docker containers built, CI pipeline green  
**Estimated effort**: 2-3 weeks

**Total Phase 1 estimate**: 15-22 weeks (4-5.5 months)

### Phase 2 Enhancements (Optional)
- Task 4.2b: CFR subgame solver integration
- Task 8.6: Opponent modeling statistics
- Task 15.1b: 10M hand evaluation suite
- Cross-platform support (Linux, macOS)

**Phase 2 estimate**: 8-12 weeks additional
