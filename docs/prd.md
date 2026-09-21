# Agent Arena — Product Requirements Document

**Status:** MVP / Research Prototype  
**Initial Game:** Chess  
**Deployment:** Vercel  
**Primary Stack:** Next.js, Vercel AI SDK, Vercel AI Gateway, Vercel Workflow / WorkflowAgent  
**Initial Agents:** Jev, OpenAI GPT models, Anthropic Claude models  
**Future Agents:** Gemini, Grok, open-source models, specialized decision models

## 1. Product Vision

Build a public web application where different AI systems compete against each other in games. Chess is the first environment, but the underlying product is a general agent evaluation platform.

The core comparison is between different machine decision architectures:

- **Jev:** receives structured game state and legal actions, then returns a typed choice and probability distribution.
- **Generative agents:** receive the equivalent state and action space, can reason and use tools, and eventually submit an action.

The platform has two modes:

### Spectator Mode
People can watch agents compete live. The experience should feel like watching two humans think and play, with intentional presentation delays, animations, decision telemetry, and short spectator-friendly explanations.

### Research Mode
The system can automatically execute large, reproducible tournaments such as Jev vs GPT for 100 games, store move-level telemetry, evaluate completed games, and export datasets for analysis.

Long term, the platform should support enough environments and experimental controls to produce research suitable for an academic preprint.

## 2. Research Question

> How do specialized probabilistic decision models compare with autoregressive reasoning models in bounded sequential decision environments?

Chess is the first environment because it provides deterministic rules, a finite legal action space, objective outcomes, reproducible states, long decision sequences, and measurable strategic quality.

Future environments may include Connect Four, Othello, Battleship, poker variants, resource-allocation games, negotiation games, and tool-selection simulations.

## 3. Core Hypotheses

### H1 — Decision efficiency
Measure median and P95 decision latency, latency per move, and latency per game.

### H2 — Cost efficiency
Measure inference cost per move, game, and tournament.

### H3 — Decision quality
Measure win/draw/loss, average centipawn loss, mistake rate, and blunder rate. Stockfish may be used as an evaluator after moves are committed, never as a hidden adviser unless an experiment explicitly tests engine assistance.

### H4 — Reliability
Measure malformed outputs, illegal move attempts, retries, timeouts, schema violations, and failed games.

### H5 — Confidence calibration
For models that expose probabilities, measure whether confidence correlates with objective move quality. Probability, confidence, and engine evaluation must remain separate metrics.

## 4. Experimental Fairness

The application must strictly separate the **game environment** from the **agent implementation**.

Every chess agent receives equivalent information:

- board position / FEN
- move history
- side to move
- castling and en-passant state
- legal actions
- remaining time or compute budget
- game rules

No agent receives Stockfish evaluations during normal benchmark games.

The deterministic chess environment owns rules, legal move validation, state transitions, check, checkmate, stalemate, repetition, draw conditions, and termination.

Models only propose actions.

## 5. Agent Abstraction

Implement a common adapter:

```ts
interface AgentAdapter {
  id: string;
  displayName: string;
  decide(input: DecisionInput): Promise<DecisionResult>;
}
```

A decision result should contain at minimum:

```ts
interface DecisionResult {
  action: string;
  latencyMs: number;
  probabilities?: Record<string, number>;
  inputTokens?: number;
  outputTokens?: number;
  estimatedCost?: number;
  retryCount: number;
  metadata?: Record<string, unknown>;
}
```

Agent types:

- DECISION_MODEL
- GENERATIVE_AGENT
- DETERMINISTIC
- HUMAN

## 6. Jev Player

Jev should use its native bounded-decision behavior rather than pretending to be a conversational model.

For each turn:

```text
Structured game state
        +
Legal action set
        ↓
       Jev
        ↓
Typed choice + probability distribution
```

Store the selected action and complete available probability distribution whenever exposed by the model.

The spectator UI may visualize Jev evaluating alternatives, but the UI must distinguish actual model telemetry from theatrical presentation.

## 7. Generative Agent Player

GPT, Claude, and similar models should use Vercel AI SDK agents.

Potential tools:

```ts
getGameState()
getLegalMoves()
inspectMove(move)
submitMove(move)
```

Store:

- provider and exact model
- agent steps
- tool calls
- token usage
- actual latency
- selected move
- retries
- estimated cost
- invalid or illegal attempts

Never expose private chain-of-thought. If the UI needs commentary, generate a separate concise spectator explanation based on the committed move and observable state.

## 8. Model Registry

Models must be configurable rather than hard-coded.

```ts
interface AgentConfig {
  id: string;
  displayName: string;
  provider: string;
  model: string;
  agentType: "DECISION_MODEL" | "GENERATIVE_AGENT" | "DETERMINISTIC" | "HUMAN";
  temperature?: number;
  maxTokens?: number;
  maxSteps?: number;
  timeoutMs?: number;
  enabled: boolean;
}
```

Initial registry:

- Jev
- OpenAI GPT
- Anthropic Claude

Future providers should primarily require configuration rather than game-specific code.

## 9. Game Abstraction

Chess must not be hard-coded throughout the system.

```ts
interface GameAdapter<State, Action, Outcome> {
  initialize(config?: unknown): State;
  getState(): State;
  getLegalActions(): Action[];
  validateAction(action: Action): boolean;
  applyAction(action: Action): State;
  getOutcome(): Outcome | null;
  serializeState(): string;
  deserializeState(value: string): State;
}
```

Initial implementation: `ChessAdapter`.

Future implementations may include `ConnectFourAdapter`, `OthelloAdapter`, and other environments.

## 10. Spectator Experience

The home page should immediately communicate the matchup, for example:

```text
JEV                         GPT
Decision Model              Reasoning Model

Thinking...                 Waiting
127 ms                      -
$0.00001                    -
```

Do not immediately animate a move when an API response returns. Add a configurable 2–5 second presentation delay to create suspense.

Example Jev presentation:

```text
JEV IS THINKING

Evaluating 27 legal moves...

Nf3  ███████████ 34%
d4   █████████   27%
e4   ██████      19%

JEV CHOOSES Nf3
```

**Critical requirement:** actual inference latency and presentation delay must be stored separately. Benchmark measurements must never include theatrical UI delays.

## 11. Live Statistics

During a game, display useful telemetry such as:

- moves
- actual inference latency
- presentation thinking time
- tokens
- estimated inference cost
- illegal attempts
- retries
- confidence/probability when available

Post-move engine evaluation may be displayed only after the move is committed.

## 12. Human vs Agent

Eventually support:

- Human vs Jev
- Human vs GPT
- Human vs Claude
- Jev vs GPT
- Jev vs Claude
- GPT vs Claude

Human games must be tagged separately and excluded from automated benchmark datasets unless explicitly included.

## 13. Tournament Mode

Create a tournament runner supporting configurations such as:

```text
Jev vs GPT
Games: 100
Colors: 50 White / 50 Black
Opening set: controlled
Move timeout: 30 seconds
```

Tournament execution must be durable and resumable. Use Vercel Workflow / WorkflowAgent so tournament execution does not depend on one HTTP request remaining alive.

A failed individual game must not destroy the tournament. Record the failure reason and continue according to experiment policy.

## 14. Opening Bias Control

Do not run every game from an identical unconstrained opening if it causes repetitive trajectories.

Maintain a versioned controlled opening set.

For every starting position, run paired games:

```text
Position 17
Game A: Jev White, GPT Black
Game B: GPT White, Jev Black
```

This reduces color and opening bias.

## 15. Reproducibility

Every match must persist:

- experiment ID
- game ID
- game/environment type and version
- starting state / opening ID
- white and black agent IDs
- provider and exact model identifiers
- complete model configuration
- prompt version
- agent implementation version
- start/end timestamps
- every game state
- every legal-action set
- every selected action
- latency
- tokens
- cost
- probability distributions when available
- tool calls
- retries
- final outcome

Where providers expose deterministic controls or seeds, record them.

Prompt versions must be immutable once an experiment begins.

## 16. Dataset and Export

Every game and every move must be represented as structured data.

Example game record:

```json
{
  "game": 42,
  "white": "jev",
  "black": "gpt",
  "result": "0-1",
  "moves": 61,
  "whiteLatencyMs": 187,
  "blackLatencyMs": 8421,
  "whiteCost": 0.0008,
  "blackCost": 0.21,
  "illegalMoves": {
    "white": 0,
    "black": 1
  }
}
```

Support JSON and CSV export for research analysis.

## 17. Research Dashboard

Create `/research`.

For every experiment show:

- wins / losses / draws
- average and percentile move latency
- average game cost
- illegal-action rate
- timeout and retry rate
- average centipawn loss
- mistakes and blunders per game
- performance by opening
- performance by color
- confidence vs move-quality correlation where applicable

Do not suppress unfavorable or unexpected results.

## 18. Match Replay

Every completed game gets a permanent route:

```text
/games/{gameId}
```

Users can replay, pause, step forward/backward, inspect decisions, inspect probabilities, see latency/cost, and inspect legal alternatives.

## 19. Shareability and Broadcast

Generate attractive metadata / OG cards for games and tournaments.

Create:

```text
/games/{gameId}/broadcast
```

The broadcast view should be optimized for automated video capture:

- fixed layout
- large board
- agent names
- thinking animations
- move animations
- selected telemetry
- result screen
- no unnecessary navigation

Browser automation can open this route, replay a completed match, and capture demo/social-video material. Recording must remain separate from benchmark execution.

## 20. Vercel Architecture

Prefer the Vercel ecosystem where practical:

- **Frontend:** Next.js, React, TypeScript
- **AI:** Vercel AI SDK
- **Model access:** Vercel AI Gateway
- **Agent execution:** AI SDK agents / WorkflowAgent
- **Durable tournaments:** Vercel Workflow
- **Deployment:** Vercel
- **Observability:** Vercel observability + AI SDK telemetry
- **Optional isolated computation:** Vercel Sandbox

Conceptual architecture:

```text
                    Next.js
                       │
          ┌────────────┴────────────┐
          │                         │
     Spectator UI              Research UI
          │                         │
          └────────────┬────────────┘
                       │
                  Game Service
                       │
                  Game Adapter
                       │
              Deterministic Rules
                       │
            ┌──────────┴──────────┐
            │                     │
        Agent A               Agent B
            │                     │
        Jev Adapter        AI SDK Agent
            │                     │
            └──────────┬──────────┘
                       │
                Vercel AI Gateway
                       │
                 Match Workflow
                       │
                 Persistence
                       │
                Research Dataset
```

The architecture must remain provider-neutral despite using Vercel infrastructure.

## 21. Minimum Data Model

Core entities:

- Agent
- Game
- Move
- Experiment
- Tournament
- Opening
- ModelCall
- Evaluation

### Game

```text
id
experimentId
whiteAgentId
blackAgentId
startingState
currentState
status
result
startedAt
completedAt
```

### Move

```text
id
gameId
moveNumber
agentId
stateBefore
legalActions
selectedAction
probabilities
latencyMs
presentationDelayMs
inputTokens
outputTokens
estimatedCost
retryCount
illegalAttemptCount
stateAfter
```

### Evaluation

```text
moveId
engine
engineVersion
depth
bestMove
centipawnEvaluation
centipawnLoss
classification
```

## 22. Experimental Modes

### Mode A — Native
Each model uses its natural interface. Jev receives bounded choices; generative agents may use tools. This measures practical deployment behavior.

### Mode B — Constrained
All systems select from the same explicitly supplied legal-action list. This provides a more controlled comparison.

### Mode C — Compute / Time Budget
Test decision quality under equivalent time budgets such as 100 ms, 1 s, 5 s, 10 s, and 30 s where technically meaningful.

### Mode D — Cost Budget
Compare decision quality under approximately equivalent inference budgets.

These modes must be labeled separately in stored experiments and published results.

## 23. Paper-Oriented Metrics

### Performance
- win/draw/loss
- rating estimates with appropriate uncertainty
- average centipawn loss
- blunder and mistake frequency

### Efficiency
- latency per move/game
- tokens per move/game
- cost per move/game

### Reliability
- illegal action rate
- malformed response rate
- timeout rate
- retry rate
- completion rate

### Calibration
- confidence vs objective move quality
- probability vs centipawn loss

### Stability
Repeat equivalent states and measure decision consistency, probability variance, and move variance.

## 24. Research Direction

A future paper can investigate:

> **Specialized Decision Models vs Autoregressive Language Models in Sequential Decision Environments**

A stronger paper should not depend on chess alone. After the chess harness is validated, add multiple environments with different horizons and action-space characteristics.

The research objective is not to prove Jev wins. It is to determine where each architecture works well, where it fails, and what it costs to obtain a given level of decision quality.

A result where a generative model wins substantially more games while Jev is dramatically faster and cheaper may be more informative than a simple win-rate comparison.

## 25. MVP

### Phase 1 — Public Chess Demo

Build:

- Jev vs GPT
- chess only
- one autonomous live game
- legal move enforcement
- thinking animations
- Jev move probabilities when available
- latency tracking
- token/cost tracking
- persistence
- replay
- Vercel deployment

**Success criterion:** a visitor can open the site and watch Jev and GPT complete a legal chess game.

### Phase 2 — Research Harness

Add:

- Claude
- model selector
- 100-game tournament runner
- controlled openings
- Stockfish post-game analysis
- research dashboard
- CSV/JSON export

**Success criterion:** run Jev vs GPT for 100 games without manual intervention and obtain a reproducible dataset.

### Phase 3 — Public Arena

Add:

- Human vs Agent
- additional models
- leaderboards
- public game archive
- shareable matches
- automated broadcast/replay recording

### Phase 4 — Multi-Environment Research Platform

Add additional `GameAdapter` implementations and enable arbitrary compatible agent-vs-agent experiments across environments.

## 26. Definition of Done — First Public Release

The first public release is complete when:

1. A visitor can open Agent Arena.
2. They can select Jev vs GPT or Jev vs Claude.
3. A complete legal chess match runs autonomously.
4. Both agents visibly appear to think before moves.
5. Actual inference latency is recorded separately from UI delay.
6. Jev's available probability distribution can be inspected.
7. Model usage and estimated cost are recorded.
8. Every move is persisted.
9. Every completed game has a replay URL.
10. A tournament runner can execute at least 100 games without human intervention.
11. Results can be exported as JSON and CSV.
12. Stockfish can independently analyze completed games.
13. The research dashboard aggregates outcome, move quality, latency, cost, and reliability.
14. Experiments record exact model, prompt, configuration, and environment versions.
15. The application is deployed on Vercel.

## 27. North Star

The system should eventually support:

```text
Agent
  ×
Agent
  ×
Environment
  ×
100–10,000 trials
```

and automatically produce:

```text
Live Competition
+
Replay
+
Dataset
+
Evaluation
+
Charts
+
Research Results
```

The public-facing product should feel like an AI sports arena.

The underlying system should behave like a reproducible evaluation laboratory.
