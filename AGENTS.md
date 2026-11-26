# Code Style

> Context is finite. Every token competes for limited attention. Maximize signal, minimize noise. Write for humans with limited working memory and AI agents with bounded context windows.

## Philosophy

- **Self-Documenting**: Names eliminate need for comments. 
- **Aggressive Minimalism**: Simplest correct solution. No speculative abstractions.
- **AHA Over DRY**: Avoid Hasty Abstractions and wait for the 3rd duplication before extracting.
- **Progressive Disclosure**: Layer code so readers grasp high-level flow immediately, drilling into details only when needed. Files answer "what is this?" in the first 10 lines. Always order code like: Types → Public API → Private helpers.

## Naming

- Use specific and verbose code: `activeUsers` not `users`, `httpTimeoutMs` not `timeout` or `to`.
- Include units: `delayMs`, `maxRetries`.
- No abbreviations: `customer` not `cust`, `configuration` not `cfg`.
- Use domain language over technical abstractions.
- Boolean prefixes: `isValid`, `hasPermission`, `canEdit`, `shouldRetry`.
- Verbs for functions: `validateEmailFormat()` not `checkEmail()`.

## Functions

- Single responsibility, describable in one sentence
- All dependencies as parameters — no hidden global state
- Type everything — strict mode, no `any`
- Guard clauses over nesting — handle edge cases first, keep happy path unindented
- 50-line guideline — not a hard limit, but a refactoring trigger
- Self-contained context units — comprehensible without reading other files

## Error Handling

- Use `Result<T, E>` types for expected failures (API calls, I/O, validation). Reserve exceptions for truly unrecoverable situations.
- Use branded types (`ValidatedEmail`, `UserId`) to validate at boundaries and carry proof through the type system.
- Never silently swallow errors — log or propagate.
- Fail fast at boundaries — validate inputs immediately.
- Provide actionable messages: what failed, expected vs actual, how to fix.

## File & Module Organisation

- Group by feature/domain, not file type: `authentication/`, `orders/`, `payments/`
- Public API first, helpers at bottom. One major export per file.
- Co-locate tests: `UserService.test.ts` next to `UserService.ts`
- 400-line guideline per file — refactoring trigger, not hard limit.
- Minimal cross-module dependencies — clean context boundary for every module.
- Use section comments (`// === PUBLIC API ===`) to create visual hierarchy in files.

## Linting & Formatting

- Use Prettier/Black/Ruff for opinionated code formatting, depending on the language.
- Linting can use, but not limited to: Biome, gofmt, rustfmt, mypy, eslint.
- Harden linting early in the project - Up-front beats retrofitting.
- Enforce linting & formatting with Git hooks.

## Testing

- Testing trophy: mostly integration tests. Static analysis → unit → integration (widest) → E2E (critical journeys only).
- Test behaviour, not implementation. One concept per test.
- Test names describe scenarios: `test('user can add items to cart')`
- 80% coverage minimum, focused on critical paths.

## Observability

- Structured logging only — JSON with `request_id`, `user_id`, `duration_ms`, entity IDs.
- Log at critical boundaries: external APIs, database ops, auth decisions, errors.
- One structured event per operation — derive metrics, logs, and traces from the same data.

## Agentic Patterns

- **Idempotent operations**: Use `ensure*` patterns. Every mutation must be safely repeatable.
- **Explicit state machines**: Model distinct phases as discriminated unions, not implicit flags.
- **Machine-parseable errors**: Discriminated unions with `code`, `message`, `retryable` fields. Agents need structured errors, not just strings.
- **Atomic changes**: Each function/commit independently testable and verifiable without understanding the entire system.
- **Convention over configuration**: Consistent naming (`*.test.ts`, `*.types.ts`), predictable directory structure, standard CRUD patterns. Reduce the search space.
- **Contract-first design**: Define types before implementation. Types are the cheapest documentation — an agent reading types understands data flow without reading function bodies.
- **Observable side effects**: Mutations return `{ data, changes[], warnings[] }` so agents can verify their actions.

## Context Optimisation

Every token your code consumes in an agent's context window is a token it can't use for reasoning.

- **Semantic compression**: Consolidate related tools/functions behind single dispatchers that route by intent. N granular tools → 1 dispatcher = massive token savings.
- **Layered context loading**:  Progressive disclosure: Summaries first, drill-down on demand. Never dump full project state upfront.
- **Token-aware docs**: Dense and scannable. No filler prose. `"JWT auth with refresh rotation"` not `"The authentication module is responsible for handling all aspects of user authentication within our application."`
- **Structured output**: Functions consumed by agents return typed objects, not human-readable prose strings.
- **Context boundaries as architecture**: Design modules so an agent can work within one module without loading adjacent modules. Co-locate types, minimise cross-module imports.
- **Paginate by default**: Any function that could return large result sets supports `limit`/`offset`.

## Project Navigation

- **AGENTS.md at root**: Entry points, key patterns, common tasks. Under 200 lines.
- **CLAUDE.md at root**: This should just reference AGENTS.md. One line max.
- **README.md per major module**: Purpose, key decisions, gotchas.
- **Progressive context hierarchy**: Root README → Module README → Section comments → Function docs → Inline comments. Each level self-sufficient.

## Anti-Patterns to avoid

- Premature optimisation
- Hasty abstractions
- Clever code
- Silent failures
- Vague interfaces (`any`),
- Hidden dependencies
- Nested conditionals
- Comments describing "what,"
- Premature generalisation
- Token bloat
- Inverted file disclosure
- Flat files without hierarchy
- Leaky context boundaries
- Eager context loading

## Checklist

- [ ] Minimal code solving the stated problem?
- [ ] Understandable without extensive surrounding context?
- [ ] Errors handled with actionable messages?
- [ ] Names clear, specific, unambiguous?
- [ ] Single responsibility per function?
- [ ] Explicit dependencies, no hidden state?
- [ ] Critical paths tested?
- [ ] Idempotent where applicable?
- [ ] Types define contracts before implementation?
- [ ] Module works independently without loading siblings?
- [ ] Public API scannable in under 50 lines?
- [ ] Documentation token-dense, no filler?
- [ ] File follows progressive disclosure (types → API → helpers)?
