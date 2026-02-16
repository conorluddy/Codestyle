# Code Style Guide

> **Core Principle**: Context is finite. Every token — code, comment, structure — competes for limited attention. Maximize signal, minimize noise. Write for two audiences: humans with limited working memory and AI agents with bounded context windows.

## Philosophy

The optimal code is the minimum necessary to solve the problem correctly. Every additional line is debt.

**Progressive Disclosure**: Structure code layer-by-layer. Readers grasp high-level flow immediately, drilling into details only when needed. File names indicate purpose. Directory structures mirror conceptual hierarchies. Function names describe behavior without reading implementation. See [Progressive Disclosure](#progressive-disclosure) for concrete patterns.

**Self-Documenting**: Names eliminate need for comments. Comments explain "why," never "what." If you chose algorithm A over B for subtle reasons, state that. If you're working around a library bug, explain it.

**Aggressive Minimalism**: Before adding code, ask: "Is this the simplest solution?" Before adding a comment: "Does this clarify something non-obvious?" Before introducing an abstraction: "Does this reduce complexity, or merely relocate it?"

**AHA Over DRY**: Avoid Hasty Abstractions. Wait for the 3rd duplication before extracting. The wrong abstraction is worse than duplication. Three similar lines of code is better than a premature abstraction.

## Progressive Disclosure

Structure every layer of your system so readers — human or agent — get the right level of detail at the right time. No one should need to read 2000 lines to understand what a module does.

### The Zoom Principle

Code should work like a map: zoom out for the big picture, zoom in for street-level detail. Each zoom level should be self-sufficient.

```
// Level 0: Directory structure tells you what exists
src/
├── authentication/     # "There's an auth system"
├── orders/             # "There's an order system"  
├── payments/           # "There's a payment system"
└── README.md           # How they connect

// Level 1: Index file tells you what it can do
// authentication/index.ts
export { authenticateUser } from './authenticate';
export { refreshSession } from './sessions';
export { revokeAccess } from './revoke';
// No implementation visible — just capabilities

// Level 2: Function signature tells you the contract
async function authenticateUser(
  credentials: UserCredentials,
  db: Database,
  clock: Clock
): Promise<Result<AuthSession, AuthError>>

// Level 3: Implementation tells you how
// Only read this when you need to change the behaviour
```

### File-Level Disclosure

Every file should answer "what is this?" in its first 10 lines. Implementation details belong below.

```typescript
// ✅ Top of file reveals purpose, contract, and shape
/**
 * Order Processing Pipeline
 * 
 * Validates → enriches → prices → submits orders.
 * Entry point: processOrder()
 * Error strategy: Result types, no throws
 */

// Types first — the contract
type ProcessOrderInput = { /* ... */ };
type ProcessOrderResult = Result<Receipt, ProcessError>;

// Public API second
export async function processOrder(input: ProcessOrderInput): Promise<ProcessOrderResult> {
  const validated = validateOrder(input);
  if (!validated.ok) return validated;
  
  const enriched = await enrichWithInventory(validated.value);
  if (!enriched.ok) return enriched;
  
  return submitOrder(enriched.value);
}

// Private helpers last — only read if you need to understand a specific step
function validateOrder(input: ProcessOrderInput): Result<ValidatedOrder, ProcessError> {
  // ...
}
```

```typescript
// ❌ Implementation soup — must read everything to understand anything
import { db } from '../globals';
const RETRY_COUNT = 3;
const BACKOFF_MS = 100;

function helper1() { /* ... */ }
function helper2() { /* ... */ }
// 200 lines later...
export function processOrder() { /* ... */ }
```

### Documentation Disclosure

Match documentation depth to the reader's likely intent. Most readers want "what does this do?" — very few want "why did you choose bcrypt over argon2?"

```
Level 1 — CLAUDE.md (5 seconds)
  "This is an order processing API. Entry: src/api/server.ts"

Level 2 — Module README (30 seconds)  
  "Orders go through validate → enrich → price → submit. 
   Uses Result types. Retries on transient failures."

Level 3 — Section comments (2 minutes)
  // ========================================
  // PRICING ENGINE
  // ========================================
  // Applies tiered discounts, tax rules, and currency conversion.
  // See: docs/pricing-model.md for business rules.

Level 4 — Inline "why" comments (as needed)
  // Using ceiling division here because partial units 
  // must be billed as full units per the SLA.
```

### API & Type Disclosure

Public interfaces should be scannable summaries. Implementation types stay internal.

```typescript
// ✅ Public types: minimal, focused, scannable
// orders/types.ts — what consumers need to know
export type OrderSummary = {
  id: OrderId;
  status: OrderStatus;
  total: Money;
  itemCount: number;
  createdAt: DateTime;
};

// orders/internal-types.ts — implementation detail
// Not exported. Contains pricing breakdowns, audit trails,
// intermediate computation states, retry metadata, etc.
type OrderPricingContext = { /* ... */ };
type OrderAuditEntry = { /* ... */ };
```

### Disclosure Anti-Patterns

- **Premature depth**: Putting implementation details in README files
- **Flat disclosure**: 500-line files with no visual hierarchy or grouping
- **Inverted disclosure**: Helpers at top, public API buried at bottom
- **Missing levels**: Jumping from directory listing straight to inline comments with nothing in between

## Naming

The #1 impact on readability. Good names eliminate mental translation overhead.

```
// ✅ Descriptive, unambiguous
async function validateJsonAgainstSchema(
  schema: ZodSchema,
  input: string
): Promise<ValidationResult>

function calculateExponentialBackoff(
  attemptNumber: number,
  baseDelayMs: number
): number

// ❌ Vague, abbreviated
async function valJson(s: any, i: string): Promise<any>
function calcBackoff(n: number, d: number): number
```

**Rules**:

1. **Be specific**: `activeUsers` not `users`, `httpTimeoutMs` not `timeout`
2. **Include units**: `delayMs` not `delay`, `maxRetries` not `max`
3. **Avoid abbreviations**: `customer` not `cust`, `configuration` not `cfg`
4. **Use domain language**: Names from business domain, not technical abstractions
5. **Boolean prefixes**: `isValid`, `hasPermission`, `canEdit`, `shouldRetry`
6. **Verbs for functions**: `validateEmailFormat()` not `checkEmail()`, `fetchActiveUsers()` not `getUsers()`

## Function Design

### Single Responsibility with Explicit Contracts

```
// ✅ Self-contained, explicit dependencies, typed contract
async function authenticateUser(
  credentials: UserCredentials,
  database: Database,
  currentTime: DateTime
): Promise<Result<AuthSession, AuthError>> {
  // All dependencies visible in signature
  // Return type reveals all possible outcomes
}

// ❌ Hidden dependencies, unclear contract
async function auth(data: any): Promise<any> {
  // Uses global config, modifies global state
}
```

### Guard Clauses Over Nesting

Handle edge cases first, keep the happy path unindented and visible.

```
// ✅ Guard clauses — happy path clear
function processOrder(order: Order): Result<Receipt, ProcessError> {
  if (!order) return err('missing_order');
  if (order.items.length === 0) return err('empty_order');
  if (order.total <= 0) return err('invalid_total');
  if (!order.paymentMethod) return err('missing_payment');

  return ok(completePayment(order));
}

// ❌ Nested conditions — happy path buried
function processOrder(order: Order) {
  if (order) {
    if (order.items.length > 0) {
      if (order.total > 0) {
        // Happy path buried 4 levels deep
      }
    }
  }
}
```

### Design Rules

1. **Single responsibility** — describable in one sentence
2. **Explicit dependencies** — all inputs as parameters, no hidden global state
3. **Type everything** — TypeScript strict mode, Python type hints
4. **Self-contained context units** — comprehensible without reading other files
5. **50-line guideline** — not a hard limit, but a refactoring trigger

## Error Handling

### Result Types — Make Errors Explicit

Errors belong in function signatures, not hidden behind `throw`.

```
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

type UserError = 'not_found' | 'unauthorized' | 'network_failure';

async function fetchUser(id: string): Promise<Result<User, UserError>> {
  // Errors are part of the contract
}

// Usage forces error handling — compiler catches missing cases
const result = await fetchUser(userId);
if (!result.ok) {
  switch (result.error) {
    case 'not_found': return show404();
    case 'unauthorized': return redirectLogin();
    case 'network_failure': return showRetry();
  }
}
```

**When to use Result types**: API calls, file I/O, validation, any complex error path.
**When to use exceptions**: Truly exceptional/unrecoverable situations (out of memory, corrupted state).

### Branded Types — Validate at Boundaries

```
type ValidatedEmail = string & { readonly __brand: 'ValidatedEmail' };
type UserId = string & { readonly __brand: 'UserId' };

function validateEmail(input: string): ValidatedEmail | null {
  return isValidEmail(input) ? (input as ValidatedEmail) : null;
}

// Type system prevents using unvalidated data
function sendEmail(to: ValidatedEmail, subject: string) {
  // No need to re-validate — type guarantees validity
}
```

Once you have a `ValidatedEmail`, downstream functions carry zero validation overhead. The type system encodes the knowledge that validation occurred.

### Error Principles

1. **Never silently swallow errors** — log or propagate, never ignore
2. **Fail fast at boundaries** — validate inputs immediately, not deep in call stack
3. **Provide actionable messages** — what failed, expected vs actual, how to fix

```
// ✅ Actionable error with context
throw new ValidationError(
  `Email validation failed for "user_email": ` +
  `Expected "name@domain.com", received "${input}". ` +
  `Use validateEmailFormat() to check before calling.`
);

// ❌ Opaque
throw new Error("Validation failed");
```

## File & Module Organization

### Structure with Clear Boundaries

```
// ========================================
// PUBLIC API
// ========================================

export class UserService {
  constructor(private readonly db: Database) {}

  async createUser(data: CreateUserData): Promise<Result<User, CreateError>> {
    // Public interface
  }
}

// ========================================
// VALIDATION
// ========================================

function validateUserData(data: unknown): Result<ValidatedData, ValidationError> {
  // Grouped validation logic
}

// ========================================
// PRIVATE HELPERS
// ========================================

function hashPassword(password: string): Promise<HashedPassword> {
  // Internal implementation
}
```

### Organization Rules

1. **Group by feature/domain**, not file type — `authentication/`, `orders/`, `payments/`
2. **Public API first** — exported functions at top, helpers at bottom
3. **One major export per file** — `UserService.ts` exports `UserService`
4. **Co-locate tests** — `UserService.test.ts` next to `UserService.ts`
5. **300-line guideline** — not a hard limit, but a refactoring trigger
6. **Minimal cross-module dependencies** — each module is a clean context boundary

```
project/
├── authentication/     # Self-contained context
│   ├── index.ts       # Public API only
│   ├── credentials.ts
│   ├── sessions.ts
│   └── README.md      # Module architecture
├── orders/            # Independent context
└── storage/           # Independent context
```

## Testing

### Testing Trophy — Mostly Integration

"Write tests. Not too many. Mostly integration." — Kent C. Dodds

1. **Static Analysis** (foundation): TypeScript strict mode, ESLint
2. **Unit Tests** (narrow): Pure functions, complex algorithms
3. **Integration Tests** (widest — most tests here): How pieces work together, where bugs actually live
4. **E2E Tests** (top): Critical user journeys only

### Tests as Documentation

Test names describe scenarios. Docstrings explain "why." Tests demonstrate usage.

```
test('should reject invalid credentials without revealing if username exists', async () => {
  // Prevents username enumeration attacks
  const auth = new Authenticator(database);

  const result = await auth.authenticate({
    email: 'nonexistent@example.com',
    password: 'any-password'
  });

  expect(result.ok).toBe(false);
  expect(result.error.code).toBe('INVALID_CREDENTIALS');
  expect(result.error.message).not.toContain('user not found');
});
```

### Testing Rules

1. **Test behavior, not implementation** — focus on inputs/outputs, not internal state
2. **One concept per test** — don't test multiple unrelated things
3. **Integration over unit** — test pieces working together (more confidence per test, more resilient to refactoring)
4. **Clear test names** — describe the scenario: `test('user can add items to cart')`
5. **80% coverage minimum** — focus on critical paths

## Observability

### Structured Logging

```
// ✅ Structured — queryable, correlated
logger.info('Request processed', {
  request_id: requestId,
  user_id: userId,
  endpoint: req.path,
  method: req.method,
  duration_ms: duration,
  status_code: res.statusCode,
  cache_hit: cacheHit
});

// ❌ Unstructured — hard to query
logger.info(`User ${userId} accessed ${req.path}`);
```

### What to Log

**Always include**: request\_id, user\_id, trace\_id, entity IDs, operation type, duration\_ms, error details.

**Log at critical boundaries**:

* External API calls (request/response)
* Database operations (query, duration)
* Authentication/authorization decisions
* Error occurrences with full context

**One structured event per operation** — derive metrics, logs, or traces from the same data. Don't instrument separately for each observability pillar.

## Agentic Coding Patterns

These patterns address the unique demands of code that will be read, modified, and executed by AI agents alongside humans.

### Idempotent Operations

Agents retry. Network calls fail. Tasks get re-run. Design every mutation to be safely repeatable.

```
// ✅ Idempotent — safe to retry
async function ensureUserExists(
  email: ValidatedEmail,
  db: Database
): Promise<User> {
  const existing = await db.users.findByEmail(email);
  if (existing) return existing;
  return db.users.create({ email });
}

// ❌ Non-idempotent — duplicates on retry
async function createUser(email: string, db: Database): Promise<User> {
  return db.users.create({ email });
}
```

### Explicit State Machines Over Implicit Flows

When operations have distinct phases, model them explicitly. Agents reason about state machines far better than implicit status flags scattered across objects.

```
type OrderState =
  | { status: 'draft'; items: Item[] }
  | { status: 'submitted'; items: Item[]; submittedAt: DateTime }
  | { status: 'paid'; items: Item[]; submittedAt: DateTime; paymentId: string }
  | { status: 'shipped'; items: Item[]; trackingNumber: string };

// Each transition is a pure function with clear preconditions
function submitOrder(order: OrderState & { status: 'draft' }): OrderState & { status: 'submitted' } {
  return { ...order, status: 'submitted', submittedAt: DateTime.now() };
}
```

### Machine-Parseable Errors

Agents need structured errors alongside human-readable ones. Return error codes that can be programmatically matched, with messages that explain context.

```
type AppError = {
  code: 'VALIDATION_FAILED' | 'NOT_FOUND' | 'CONFLICT' | 'UPSTREAM_TIMEOUT';
  message: string;        // Human-readable explanation
  field?: string;         // Which input caused it
  retryable: boolean;     // Can the caller retry?
};
```

### Atomic, Independently-Verifiable Changes

Structure work so each change can be validated in isolation. This applies to commits, PRs, and function design. An agent (or reviewer) should be able to verify correctness without understanding the entire system.

```
// ✅ Each function is independently testable and verifiable
function parseConfig(raw: string): Result<Config, ParseError> { /* ... */ }
function validateConfig(config: Config): Result<ValidConfig, ValidationError[]> { /* ... */ }
function applyConfig(config: ValidConfig, system: System): Result<void, ApplyError> { /* ... */ }

// ❌ Monolithic — must understand everything to verify anything
function loadAndApplyConfig(path: string): void { /* 200 lines */ }
```

### Convention Over Configuration

Reduce the search space for agents (and humans). Consistent patterns mean less context needed per decision.

* Consistent file naming: `UserService.ts`, `UserService.test.ts`, `UserService.types.ts`
* Predictable directory structure across features
* Standard patterns for CRUD operations, API endpoints, error handling
* If your project has a pattern, follow it. If it doesn't, establish one and document it

### Contract-First Design

Define types before implementation. Types are the cheapest, most scannable form of documentation. An agent reading your types understands your system's data flow without reading a single function body.

```
// Define the contract first
interface OrderService {
  create(data: CreateOrderInput): Promise<Result<Order, CreateOrderError>>;
  cancel(id: OrderId, reason: CancelReason): Promise<Result<void, CancelError>>;
  findByUser(userId: UserId, pagination: Pagination): Promise<PaginatedResult<OrderSummary>>;
}

// Then implement — the types guide everything
```

### Observable Side Effects

Every mutation should produce structured output describing what changed. This enables agents to verify their actions and enables humans to audit.

```
type MutationResult<T> = {
  data: T;
  changes: Change[];  // What was modified
  warnings: string[]; // Non-fatal issues encountered
};

async function updateUserProfile(
  id: UserId,
  updates: ProfileUpdates
): Promise<Result<MutationResult<UserProfile>, UpdateError>> {
  // Returns both the result AND a description of what changed
}
```

### Context Optimisation & Token Economics

> Every token an agent reads is a token it can't use for reasoning. Treat context like memory in an embedded system — budget it, measure it, and refuse to waste it.

#### The Context Budget

AI agents operate within fixed context windows. Your code, documentation, error messages, and tool outputs all compete for the same finite space. Code that is token-efficient isn't just neat — it directly improves agent reasoning quality.

```
Context Window (finite)
├── System prompt & instructions     ~2-5k tokens (fixed cost)
├── Conversation history             ~variable
├── Tool definitions                 ~1-10k tokens (per tool schema)
├── Retrieved code / docs            ~variable ← YOU CONTROL THIS
├── Agent reasoning                  ~variable ← THIS GETS SQUEEZED
└── Output generation                ~variable ← AND SO DOES THIS

The more tokens your code consumes, the less room the agent 
has to think. Optimise ruthlessly.
```

#### Semantic Compression

Collapse granular interfaces into high-level semantic operations. Instead of exposing every low-level action, expose intent-based APIs.

```typescript
// ❌ 15 granular tools = ~15k tokens of schema
// An agent must read and reason about ALL of them
tools: [
  createFile, readFile, deleteFile, moveFile, copyFile,
  listDirectory, createDirectory, deleteDirectory,
  getFileMetadata, setFilePermissions, watchFile,
  compressFile, decompressFile, hashFile, diffFiles
]

// ✅ 1 semantic dispatcher = ~1k tokens of schema
// Agent reasons about intent, not mechanics
tools: [{
  name: "filesystem",
  description: "Manage files and directories",
  parameters: {
    operation: "create | read | delete | move | copy | list | ...",
    path: "string",
    options: "object (operation-specific)"
  }
}]
```

This is the dispatcher pattern: consolidate related tools behind a single entry point that routes by intent. Token cost drops dramatically while functionality stays the same.

#### Layered Context Loading

Don't front-load everything. Provide summaries first, with drill-down paths for when the agent actually needs more detail.

```typescript
// ✅ Layered: summary first, details on demand
function getProjectOverview(): ProjectSummary {
  return {
    name: "DataPipeline",
    modules: ["ingestion", "transform", "export"],
    entryPoint: "src/main.ts",
    recentChanges: getRecentChangeSummary(5),
    // Drill-down references — agent only loads what it needs
    getModuleDetail: (name: string) => loadModuleContext(name),
    getFileContent: (path: string) => loadFileContext(path),
  };
}

// ❌ Eager: dumps everything into context upfront
function getProjectContext(): FullProjectDump {
  return {
    allFiles: readAllFiles(),          // 50k tokens
    allTests: readAllTests(),          // 30k tokens
    allDocs: readAllDocs(),            // 20k tokens
    // Agent's context window is now full before it starts thinking
  };
}
```

#### Token-Aware Documentation

Write documentation that serves both human readers and token budgets. Every word should earn its place.

```markdown
# ❌ Token-heavy: narrative style, repetitive, verbose
## Overview of the Authentication Module
The authentication module is responsible for handling all aspects of user 
authentication within our application. This module was designed with security 
best practices in mind and implements industry-standard protocols. The module 
handles user login, token generation, session management, and token refresh 
functionality. It is important to note that this module uses JWT tokens for 
authentication purposes.
(~80 tokens to say what could be said in 15)

# ✅ Token-efficient: dense, scannable, no filler
## Authentication
JWT-based auth with refresh token rotation.
- Entry: `authenticate()` → `Result<Session, AuthError>`  
- Tokens: 15min access, 7d refresh (HTTP-only cookie)
- Storage: PostgreSQL users, Redis token blacklist
(~40 tokens, more information conveyed)
```

#### Structured Output for Agent Consumption

When building tools or functions that agents will consume, prefer structured, parseable output over human-readable prose.

```typescript
// ✅ Agent-friendly: structured, parseable, minimal
type BuildResult = {
  success: boolean;
  errors: { file: string; line: number; code: string; message: string }[];
  warnings: { file: string; line: number; code: string; message: string }[];
  stats: { duration_ms: number; filesProcessed: number };
};

// ❌ Human-only: requires parsing natural language
function getBuildOutput(): string {
  return `Build completed with 2 errors and 1 warning.
    Error in src/auth.ts line 42: Type 'string' is not assignable...
    Error in src/orders.ts line 18: Property 'id' does not exist...
    Warning in src/utils.ts line 7: Unused variable 'temp'...
    Build took 3.2 seconds, processed 47 files.`;
}
```

#### Context Boundaries as Architecture

Design modules so an agent can work within one module without loading others. Each module should be a self-contained context unit.

```typescript
// ✅ Clean context boundary — agent only needs this module
// payments/index.ts
export interface PaymentService {
  charge(input: ChargeInput): Promise<Result<Payment, PaymentError>>;
  refund(id: PaymentId, reason: RefundReason): Promise<Result<Refund, RefundError>>;
}

// payments/types.ts — all types co-located, no external dependencies
export type ChargeInput = {
  amount: Money;
  method: PaymentMethod;
  idempotencyKey: string;  // Agent-friendly: built-in retry safety
};

// payments/errors.ts — exhaustive, machine-readable
export type PaymentError =
  | { code: 'INSUFFICIENT_FUNDS'; available: Money }
  | { code: 'CARD_DECLINED'; reason: string; retryable: false }
  | { code: 'GATEWAY_TIMEOUT'; retryable: true };
```

```
// ❌ Leaky context boundary — agent must load 4 modules to understand 1
// payments/index.ts
import { User } from '../users/types';
import { Order } from '../orders/types';
import { AuditLogger } from '../audit/logger';
import { ConfigManager } from '../config/manager';
// Agent now needs context from users/, orders/, audit/, config/
```

#### Compression Strategies Reference

| Strategy | Before | After | Savings |
|----------|--------|-------|---------|
| Semantic dispatchers | N tool schemas (~N × 1k tokens) | 1 dispatcher (~1k tokens) | ~(N-1)k tokens |
| Layered loading | Full dump (50k tokens) | Summary + drill-down (2k + on-demand) | ~48k idle tokens |
| Dense docs | Narrative prose (~80 tokens/concept) | Structured bullets (~40 tokens/concept) | ~50% |
| Co-located types | Scattered across modules | Single `types.ts` per module | Fewer file loads |
| Summary-first returns | Full object graphs | Summary + reference IDs | 60-90% per call |
| Discriminated unions | Generic error + message string | Typed union with `code` field | Eliminates parsing |

## Project Navigation

### CLAUDE.md at Project Root

Every project needs a navigation file. List entry points, patterns, and common tasks.

```
# Project: Data Processing Pipeline

## Entry Points
- `src/main.ts`: CLI interface
- `src/api/server.ts`: REST API
- `src/processors/pipeline.ts`: Core processing

## Key Patterns
- All processors implement `Processor` interface (src/processors/base.ts)
- Config uses Zod schemas (src/config/schemas.ts)
- External APIs via `APIClient` (src/external/client.ts)

## Common Tasks
- Add data source → implement `DataSource` in `src/api/sources/`
- Add transformation → implement `Transformer` in `src/processors/transformers/`
```

Keep under 200 lines. Update when architecture changes.

### Module-Level READMEs

Every major directory gets a README answering: What is this? How does it work? What are the gotchas?

```
# Module: User Authentication

## Purpose
JWT-based authentication with refresh token rotation

## Key Decisions
- bcrypt cost factor 12 for password hashing
- Access tokens expire after 15 minutes
- Refresh tokens stored in HTTP-only cookies

## Dependencies
- jose library for JWT (not jsonwebtoken — more secure)
- PostgreSQL for user storage
- Redis for token blacklist
```

### Progressive Context Hierarchy

1. **CLAUDE.md / README.md at root** — system overview, entry points, setup
2. **README.md per major module** — module purpose, key decisions, patterns
3. **Section comments in files** — group related code with clear headers
4. **Function/class docs** — purpose, examples for non-obvious APIs
5. **Inline comments** — only for "why" decisions

## Anti-Patterns

* **Premature optimization** — Measure first, optimize second
* **Hasty abstractions** — Wait for 3rd duplication before extracting
* **Clever code** — Simple and obvious beats clever and compact
* **Silent failures** — Log and propagate, never swallow
* **Vague interfaces** — `process(data: any): any` provides zero guidance
* **Hidden dependencies** — Global state, singletons, ambient imports
* **Nested conditionals** — Use guard clauses instead
* **Comments describing "what"** — If you need a comment to explain what code does, rename things
* **Premature generalization** — Build for today's requirements, not hypothetical futures
* **Token bloat** — Functions returning everything when callers need summaries
* **Inverted disclosure** — Helpers at top, public API buried at bottom
* **Flat files** — 500-line files with no visual hierarchy, grouping, or section comments
* **Leaky context boundaries** — Modules that import heavily from siblings, forcing agents to load the entire codebase
* **Eager context loading** — Dumping full project state into agent context when a summary would suffice

## Checklist

Before submitting code:

* Solves the stated problem with minimal code?
* A new developer can understand it without extensive context?
* Errors handled with actionable messages?
* Names clear, specific, and unambiguous?
* Functions have single, clear responsibilities?
* Dependencies explicit (no hidden global state)?
* Tests cover critical paths?
* Operations idempotent where applicable?
* Types define contracts before implementation?
* Would this work well with ~200 lines of surrounding context?
* Can an agent understand this module without loading adjacent modules?
* Are public APIs scannable in under 50 lines?
* Do tool/function outputs use structured types, not prose?
* Is documentation token-dense (no filler words, no repetition)?
* Does the file follow progressive disclosure (types → public API → helpers)?

---

*"Any fool can write code that a computer can understand. Good programmers write code that humans can understand." — Martin Fowler*
