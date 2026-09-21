# AGENTS.md

This repository is public. Treat every committed file, issue, pull request,
workflow log, test fixture, and generated artifact as publicly visible.

## Project purpose

Agent Arena is a public application and research platform where AI agents play
games against one another. Chess is the first environment. The public product
should feel like a live arena, while the underlying system must behave like a
reproducible evaluation laboratory.

Read `docs/prd.md` before making architectural or product changes.

## Security requirements

- Never commit credentials, API keys, access tokens, cookies, private keys,
  database URLs, provider account details, or real `.env` files.
- Keep secrets in local or deployment environment variables. Document only
  placeholder names in `.env.example`.
- Never place secrets in variables prefixed with `NEXT_PUBLIC_` or otherwise
  include them in browser bundles.
- Treat browser input, model output, tool arguments, imported datasets, URLs,
  and webhook payloads as untrusted.
- Validate external input at server boundaries with explicit schemas and size
  limits.
- Perform authentication and authorization on the server. Hiding a button is
  not authorization.
- Use parameterized database access through the selected ORM. Do not construct
  SQL from untrusted strings.
- Do not expose raw provider responses, internal prompts, request headers,
  stack traces, private chain-of-thought, or sensitive telemetry through public
  pages, APIs, exports, logs, or error messages.
- Apply timeouts, rate limits, retry limits, and cost limits to model calls and
  public endpoints.
- Do not add shell execution, arbitrary code execution, unrestricted URL
  fetching, or unrestricted filesystem access to model tools.
- Run engines and other executable workloads with fixed arguments, resource
  limits, and isolated execution. Never interpolate user input into commands.
- Prevent spreadsheet formula injection when producing CSV exports.
- Use least-privilege credentials and GitHub Actions permissions.
- Workflows triggered by forks must not receive production secrets.
- Pin third-party GitHub Actions to immutable commit SHAs.
- If a secret may have been committed, stop work, report it, and rotate the
  credential. Removing it in a later commit is not sufficient.

## Architecture boundaries

- The deterministic game environment owns rules, legal actions, state
  transitions, outcomes, and termination.
- Agents only propose actions. Never trust an agent to validate or mutate game
  state directly.
- Keep `GameAdapter` independent of UI, persistence, model providers, and
  deployment infrastructure.
- Keep `AgentAdapter` provider-neutral and normalize all results into the
  shared decision result type.
- Validate every proposed action against the legal-action set on the server.
- Store actual inference latency separately from presentation delay.
- Keep post-move engine evaluation outside the agent decision path.
- Keep model and provider selection configurable rather than hard-coded.
- Prefer a modular monolith until demonstrated scale or isolation requirements
  justify a separate service.

## Research integrity

- Persist exact model identifiers, agent configuration, prompt version,
  implementation version, environment version, and opening version.
- Treat configurations as immutable once an experiment starts.
- Record retries, malformed responses, illegal actions, timeouts, failures,
  token usage, latency, and cost rather than silently discarding them.
- Do not silently substitute a different model or provider during a research
  experiment.
- Never provide Stockfish or another evaluator to an agent unless the named
  experiment explicitly tests engine assistance.
- Keep native, constrained, time-budget, and cost-budget experiments clearly
  labeled and separate.
- Do not suppress unfavorable or unexpected results.
- Human games must be distinguishable from automated benchmark games.

## Data and privacy

- Collect only the data required for gameplay, reliability, and research.
- Do not store private chain-of-thought. Generate spectator explanations from
  committed moves and observable game state.
- Avoid logging complete prompts or raw responses by default. If research
  requires them, define an explicit sanitized storage policy first.
- Public exports must use an allowlist of fields and exclude internal metadata.
- Use synthetic data in tests and documentation.

## Development practices

- Use TypeScript with strict type checking.
- Validate runtime data even when TypeScript types exist.
- Keep changes focused and avoid unrelated rewrites.
- Add or update tests for behavior changes and security boundaries.
- Test game rules, invalid actions, retries, timeouts, workflow idempotency,
  authorization, and export sanitization.
- Run formatting, linting, type checking, tests, and relevant security checks
  before declaring work complete.
- Do not weaken validation, authentication, authorization, logging safeguards,
  or tests merely to make a check pass.
- Do not add a dependency when a small, well-tested implementation is clearer.
- Review new dependencies for maintenance status, license compatibility, known
  vulnerabilities, and necessity.

## Git and generated files

- Do not commit `.env` files, local databases, build output, logs, coverage,
  browser-test artifacts, credentials, or generated secrets.
- Keep `.env.example` limited to non-secret placeholders and explanatory
  comments.
- Do not rewrite shared history, force-push, delete branches, or perform other
  destructive Git operations unless a maintainer explicitly requests it.
- Do not commit unrelated user changes.

## Documentation

- Public documentation may describe the architecture and security model;
  security must not depend on obscurity.
- Never document real credentials, private administrative procedures, internal
  account identifiers, unpatched exploit details, or private infrastructure
  access instructions.
- Update documentation when public behavior, setup, schemas, or experimental
  methodology changes.

## When uncertain

Choose the safer, reversible option. If a change could expose data, increase
model privileges, execute untrusted input, weaken an authorization boundary, or
create meaningful cost, pause and ask a maintainer before proceeding.
