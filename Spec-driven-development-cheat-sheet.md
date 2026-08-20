# Spec-Driven Development Cheat Sheet

## Why Spec-Driven Development Matters

Prompt-based AI coding ("vibe coding") works well for throwaway scripts and early spikes but degrades quickly on production systems. The failure modes are predictable:

- **Intent drift**: Prompts like "add login" are underspecified — the model picks defaults (JWT or sessions? which DB schema? what error format?) that rarely match team intent.
- **Context decay**: Agents forget earlier decisions as the codebase grows across sessions, silently contradicting choices made three conversations ago.
- **Unverifiable output**: Without explicit acceptance criteria, reviewing AI-generated code becomes open-ended and unreliable.

Spec-Driven Development (SDD) addresses this by making a structured, version-controlled specification the single source of truth. The development order is: **write the spec first**, derive a plan, decompose into atomic tasks, then generate code — treating code as a build artifact of the specification rather than the primary artifact.

**The result:** Unguided AI coding succeeds roughly 33% of the time on non-trivial tasks. Teams adopting SDD report 3–10× higher first-pass success rates. Structured specifications reduce LLM-generated code errors by approximately 50% (arXiv 2602.00180). An extra hour writing a spec typically saves three days of agent thrash and three weeks of code review.

---

## SDD vs Prompt-Based Coding

```
PROMPT-BASED CODING                       SPEC-DRIVEN DEVELOPMENT
──────────────────────────────            ──────────────────────────────────────
User types vague goal                     User writes short intent brief
           │                                             │
           ▼                                             ▼
  AI generates code immediately             AI explores codebase (Plan Mode)
           │                                             │
           ▼                                             ▼
  Wrong assumptions baked in                AI drafts SPEC.md → human reviews
           │                                             │
           ▼                                             ▼
  Rounds of back-and-forth debugging        AI surfaces ambiguities via Q&A
           │                                             │
           ▼                                             ▼
  Unpredictable, hard-to-review output      AI decomposes into atomic tasks
                                                         │
                                                         ▼
                                             Subagents implement each task
                                                         │
                                                         ▼
                                             Pre-commit hooks verify at every step
                                                         │
                                                         ▼
                                             Deterministic, reviewable output
```

Prompt-based coding is faster to start but compounds debt at each iteration. SDD moves ambiguity resolution to before the first line of code. **When the agent starts asking pointed clarifying questions instead of generating code, the spec is good.**

### Three Implementation Levels

| Level | Description | When to Use |
|---|---|---|
| **Spec-First** | Write spec before AI-assisted coding; spec may drift afterward | Prototypes, AI-assisted feature spikes |
| **Spec-Anchored** | Spec evolves alongside code with human maintenance | Sweet spot for most production systems |
| **Spec-as-Source** | Code is entirely generated from spec; spec is the only maintained artifact | Well-defined domains (e.g., OpenAPI stubs, CRUD scaffolding) |

---

## The SDD Workflow

```
[Explore] ──▶ [Specify] ──▶ [Refine] ──▶ [Plan] ──▶ [Implement] ──▶ [Verify]
     │              │             │            │             │              │
  Read files,    AI drafts    AI surfaces   Ordered      Subagents       Pre-commit
  no edits       SPEC.md from  ambiguities  task list    execute each    hooks +
  (Plan Mode)    your brief;   before any   from spec    task; commit    acceptance
                 human reviews code written              after each      criteria
```

**Gate rule: never proceed to the next phase without explicit human review and approval at each boundary.** Skipping gates is vibe coding disguised as SDD.

1. **Explore** — Activate Plan Mode (Shift+Tab in Claude Code) so the agent reads without editing. It examines existing code, patterns, and relevant external libraries. No code is written in this phase.

2. **Specify** — Provide a short intent brief (1–3 sentences + constraints). Let the AI draft the full SPEC.md — agents surface assumptions in spec form better than humans do from scratch. You review, correct wrong assumptions, and save.

3. **Refine** — The AI surfaces ambiguities via structured Q&A before any code is written. This is the cheapest point to catch misalignments. The most useful specs are self-contained: they name the files and interfaces involved, state what is out of scope, and end with a verification step that proves the feature works.

4. **Plan** — The AI derives an ordered task list from the approved spec. Each task is scoped to a single logical unit of work that can be completed by a subagent in one context window.

5. **Implement** — Subagents execute each task in isolation (fresh context window per task). The main orchestrator tracks dependencies and commits after each completed task. This prevents "agent amnesia" — context saturation from accumulated prior decisions.

6. **Verify** — Pre-commit hooks run type checking, linting, and tests before any commit lands. The agent sees failures immediately and self-corrects within the same task rather than discovering regressions sessions later.

---

## Writing a Good Spec (SPEC.md)

### Full Template

````markdown
# Feature Spec: [Feature Name]
Version: 1.0 | Status: Draft | Owner: [name]

## Objective
[1–3 sentences: what does this feature do and why?]

## Not Included
- [Adjacent feature explicitly out of scope]
- [Another adjacent feature out of scope]

## Tech Stack
- Runtime: [Name, version]
- Language: [Name, version]
- Key libraries: [Name, version]

## Input / Output Contracts
```zod
// Use Zod, JSON Schema, or OpenAPI — machine-readable schemas,
// not prose descriptions. Agents infer types from schemas, not paragraphs.
const InputSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})
const OutputSchema = z.object({
  token: z.string(),
  expiresAt: z.number(),
})
```

## Functional Requirements
- WHEN user submits valid credentials, THEN system returns a signed JWT and sets HttpOnly cookie.
- WHILE session is active, system shall refresh the token 5 minutes before expiry.
- IF login fails 5 times in 10 minutes, THEN lock the account and send an email alert.

## Non-Functional Requirements
- P99 login latency < 200ms
- Token expiry: 15 minutes (access), 7 days (refresh)
- Passwords stored as bcrypt, cost factor 12

## Acceptance Criteria
| ID  | Scenario           | Input                               | Expected Output                                       |
|-----|--------------------|-------------------------------------|-------------------------------------------------------|
| AC1 | Valid credentials  | email: valid, password: correct     | `{ token: "...", expiresAt: <epoch> }`, status 200    |
| AC2 | Wrong password     | email: valid, password: incorrect   | `{ error: "INVALID_CREDENTIALS" }`, status 401        |
| AC3 | Account locked     | 5 failed attempts in 10 min         | `{ error: "ACCOUNT_LOCKED" }`, status 423             |
| AC4 | Expired token      | Expired JWT in Authorization header | `{ error: "TOKEN_EXPIRED" }`, status 401              |

## Edge Cases and Error Handling
- Unknown email → same 401 as wrong password (prevents user enumeration)
- Simultaneous login from two devices → both sessions valid; independent refresh tokens
- Error taxonomy: `INVALID_CREDENTIALS`, `ACCOUNT_LOCKED`, `TOKEN_EXPIRED`, `TOKEN_INVALID`

## Boundaries
- **Always**: Hash passwords before storage; validate all inputs before any DB call; log auth failures with trace ID
- **Ask First**: Any change to token expiry windows; disabling account lock policy
- **Never**: Log plaintext passwords; store tokens in localStorage; touch the `users_audit` table directly

## Test Plan
- Run: `pnpm test --coverage --testPathPattern=auth`
- Coverage target: ≥ 90% on `src/auth/**`
- Prohibited completion phrases: "should pass", "looks correct", "best practices"
````

### What to Exclude (Anti-Patterns)

**Implementation hints** — Instead of "Use a HashMap for the session store," say "Session lookup must be O(1)." Leave the how to the agent; specify the what and the constraint.

**Pseudo-code** — Agents treat it as literal production code. Replace pseudo-code with behavioral specifications and EARS requirements (see next section).

**Vague qualifiers** — Replace "fast" with "P99 < 200ms." Replace "secure" with specific auth requirements. Replace "covered" with an exact command and percentage target. Vague specs produce confident wrong code.

**Prescriptive architecture** — Instead of mandating class hierarchies, point to an existing pattern: "Follow the pattern in `src/payments/PaymentsService.ts`." Agents learn faster from examples in the actual codebase than from abstract hierarchies.

**Conflicting instructions** — Agents silently drop constraints when two sections conflict rather than flagging the contradiction. Cross-check all sections before handoff.

### File Organization

```
specs/
  001-user-auth/
    spec.md      ← permanent source of truth; never discard
    plan.md      ← volatile; discard after completion
    tasks.md     ← checklist; check off as tasks are executed
  002-payments/
    spec.md
    ...
```

Plans are volatile — discard them after completion. Specs are permanent — the source of truth across all future sessions. Never treat them as the same artifact.

---

## EARS Notation for Requirements

EARS (Easy Approach to Requirements Syntax) gives requirements a predictable structure that agents can parse reliably. Prose requirements ("the system should handle errors gracefully") produce vague implementations. EARS requirements produce deterministic ones.

**1. Ubiquitous** — Always-true system statements. No trigger needed.
```
The system shall store all passwords as bcrypt hashes with cost factor ≥ 12.
```

**2. Event-driven** — WHEN [trigger] THEN [system response]
```
WHEN a user submits valid credentials,
THEN the system shall return a signed JWT and set an HttpOnly cookie with the refresh token.
```

**3. State-driven** — WHILE [condition] [system behavior]
```
WHILE a session is active,
the system shall silently refresh the access token 5 minutes before expiry
without requiring re-authentication.
```

**4. Unwanted behavior** — IF [unwanted condition] THEN [prevention or recovery]
```
IF login fails 5 times within 10 minutes,
THEN the system shall lock the account and send an email notification to the registered address.
```

**5. Optional feature** — WHERE [feature exists] [system behavior]
```
WHERE two-factor authentication is enabled,
the system shall require a valid TOTP code after password verification
before issuing a session token.
```

Apply EARS consistently — every functional requirement should map to one of these five patterns. If a requirement does not fit any pattern, it is likely a non-functional requirement (performance, security, accessibility) or an implementation hint that should be removed.

---

## Acceptance Criteria

Acceptance criteria must be concrete — an input/output table, not prose. Prose criteria ("the system should handle errors gracefully") let agents interpret freely and make automated verification impossible.

```markdown
| ID  | Scenario          | Input                                     | Expected Output                                                      |
|-----|-------------------|-------------------------------------------|----------------------------------------------------------------------|
| AC1 | Happy path        | Valid Visa (4111111111111111, 12/28, 123) | `{ valid: true, errors: [] }`                                        |
| AC2 | Expired card      | Expiry date in the past (01/24)           | `{ valid: false, errors: [{ field: "expiry", code: "EXPIRED" }] }`   |
| AC3 | Invalid Luhn      | Card number fails Luhn check              | `{ valid: false, errors: [{ field: "number", code: "LUHN_FAIL" }] }` |
| AC4 | Missing CVC       | CVC field empty                           | `{ valid: false, errors: [{ field: "cvc", code: "REQUIRED" }] }`     |
```

**Prohibited completion phrases** — Add these explicitly to every spec's Test Plan section to prevent agents marking work done before running verification:
- `"should pass"` — not the same as "does pass"
- `"looks correct"` — not the same as "is correct"
- `"best practices"` — meaningless without a citation
- `"this handles the case"` — meaningless without a test run

An agent that returns these phrases without executing the test command has not verified anything.

---

## SDD vs TDD vs BDD

| Aspect | TDD | BDD | SDD |
|---|---|---|---|
| **Primary artifact** | Failing unit test | Gherkin scenario | Versioned SPEC.md |
| **Execution order** | Test → Code → Refactor | Scenario → Steps → Code | Spec → Plan → Tasks → Code |
| **Language** | Production language | Gherkin (business English) | Structured Markdown + schemas |
| **Scope** | One unit / function | One behavior / user flow | One feature (behavior + architecture + edge cases) |
| **Primary executor** | Developer | Developer + QA | Developer + AI agent |
| **When written** | Immediately before coding | Before coding | Before planning, before coding |

TDD and BDD operate at a lower altitude than SDD — they are SDD applied at the unit and behavior level respectively. SDD extends rather than replaces them: it builds on TDD at the unit level, incorporates BDD's executable specifications, and aligns with DDD's emphasis on ubiquitous language.

**Compose all three.** Run SDD at the feature level to align on what to build. Keep TDD at the unit level for fast local design feedback. Use BDD for user-facing flows (Given/When/Then with browser automation). A good SDD workflow still produces unit and integration tests — they are generated from the spec's acceptance criteria, not written afterward as an afterthought.

---

## Real-World Example

### Before (Prompt-Based)

**Prompt:**
> "Add user authentication with login and signup"

**What happens:**
- AI picks JWT over sessions — not what the team uses elsewhere in the codebase
- Stores tokens in `localStorage` — a known XSS vulnerability
- Uses a different error format than the rest of the API (`message` instead of `error`)
- No account lockout logic included
- Three rounds of corrections follow; the fourth attempt introduces a regression in a related middleware

### After (Spec-Driven)

**Intent brief to the agent (3 sentences):**
> "Add email/password authentication. We use Express + Prisma + Postgres. Sessions via HttpOnly cookie, not localStorage. Follow the error format in `src/shared/errors.ts`. Reference the existing `hashPassword` util in `src/shared/crypto.ts`."

**SPEC.md excerpt the agent drafts:**

```markdown
## Objective
Add email/password login and signup to the Express API. Sessions are managed via
HttpOnly cookies containing a signed refresh token. Access tokens expire in 15 minutes;
refresh tokens in 7 days. Error format follows `src/shared/errors.ts`.

## Not Included
- OAuth / social login
- Two-factor authentication
- Password reset flow (separate spec)

## Boundaries
- Never: store tokens in localStorage; log plaintext passwords; write to `users_audit` directly
- Ask First: any change to token expiry windows; disabling account lockout

## Acceptance Criteria
| ID  | Scenario        | Input                    | Expected Output                          |
|-----|-----------------|--------------------------|------------------------------------------|
| AC1 | Valid login     | Correct email + password | 200, signed JWT, HttpOnly cookie set     |
| AC2 | Wrong password  | Correct email, bad pass  | 401, `{ error: "INVALID_CREDENTIALS" }`  |
| AC3 | Account locked  | 5 fails in 10 min        | 423, `{ error: "ACCOUNT_LOCKED" }`       |
```

**Agent's first response (before writing a single line of code):**
> "I found an existing `hashPassword` util in `src/shared/crypto.ts` — I'll reuse that rather than adding bcrypt directly. Should I wire this into the existing `AuthMiddleware` or create a separate `LoginMiddleware`? Also, the `users` table currently has no `failedLoginCount` or `lockedUntil` column — I'll need to add a migration. Should I create it or does a separate migration spec exist?"

Two questions. Zero regressions. Zero wasted implementation. One pull request.

### Scale Result

Using the SDD workflow with Claude Code subagents on a non-trivial SQLite → IndexedDB migration (alexop.dev):
- **Time:** ~45 minutes
- **Output:** 14 commits, 15+ files changed, production-ready PR
- **Context efficiency:** main orchestrator reached 71% context usage despite 14 delegated subagents — subagent isolation kept the main window clean across the full migration

---

## Common Pitfalls

1. **Spec staleness / drift** — A stale spec is more dangerous than no spec because agents execute outdated plans confidently without flagging misalignments. Fix: bidirectional maintenance. When an agent discovers a discrepancy ("found an existing theme context provider — wired into that instead of creating a new store"), it updates the spec and reports the deviation. Shared human-agent ownership prevents rot.

2. **Over-specification** — Constraining implementation choices removes the agent's ability to make sensible decisions and blurs the line between spec (what to build) and code (how to build it). Fix: specify behavior and outcomes ("lookup must be O(1)"), not data structures ("use a HashMap").

3. **Under-specification / vague language** — Studies of large agent file datasets find most failures trace to vague instructions. Fix: every functional requirement follows one of the five EARS patterns. Every acceptance criterion has a concrete input and expected output. No prose assertions.

4. **Over-applying the methodology** — SDD adds real overhead; not every task justifies it. Fix: use SDD for large refactors, migrations, unclear requirements, or new library adoption. Skip it for simple bug fixes, single-file changes, and obvious CRUD additions.

5. **Skipping human checkpoints** — Automatically proceeding through all phases without review is vibe coding with extra steps. Fix: gate every phase boundary. Never proceed from Specify to Plan, or Plan to Tasks, or Tasks to Implement, without explicit human review and approval.

6. **Specs outside version control** — Specs stored in a wiki or doc tool rot without maintenance pressure. Developers don't update what they don't see. Fix: specs live in the same repository as code. Cite them in commits: `feat(auth): magic link, refs specs/004`.

7. **Treating specs as immutable** — If requirements change but the spec does not, every subsequent agent interaction is grounded in outdated information. Fix: update the spec after every implementation cycle. A spec that reflects what was built, not what was originally planned, is the correct source of truth.

---

## Tooling

### Claude Code

- **Plan Mode** (Shift+Tab) — activates the Explore phase; the agent reads without editing, surfacing existing patterns and naming files that will be involved before any code is written.
- **`/sdd:specify`** — produce a feature specification from a brief.
- **`/sdd:clarify`** — surface ambiguities and ask targeted questions before implementation begins.
- **`/sdd:plan`** — create a technical architecture from an approved spec.
- **`/sdd:tasks`** — decompose the plan into an ordered, trackable task checklist.
- **`/sdd:implement`** — agent executes each task with test verification before moving to the next.

**Subagent architecture:** The main orchestrator creates and tracks tasks; each task is delegated to a subagent with a fresh context window. This eliminates context saturation from accumulated prior decisions and keeps each task focused on a single logical unit of work.

### AWS Kiro

Kiro is an agentic IDE built around the SDD workflow. When you describe a feature, it does not write code immediately — it generates three documents in sequence, stored in `.kiro/specs/`:

1. **Requirements document** — user stories in EARS or BDD format, business value, acceptance criteria
2. **Design document** — technical architecture, data flow, edge cases, integration points
3. **Task list** — 5–8 sequential, trackable implementation steps

Each document requires developer review and approval before Kiro proceeds to the next phase. **Steering rules** stored in the repository govern all AI interactions — equivalent to `CLAUDE.md` in Claude Code.

### Pre-Commit Hooks as Quality Backpressure

Wire type checking, linting, and tests into pre-commit hooks so subagents see failures immediately and self-correct before a broken commit lands:

```bash
# .husky/pre-commit
pnpm typecheck && pnpm lint && pnpm test-run
```

This shifts quality validation from human code review to automated system feedback — the agent's tightest and fastest feedback loop. Agents that fail a hook self-correct within the same task rather than propagating the error to subsequent tasks.

---

## Managing Specs Day-to-Day

1. **The constitution pattern** — Encode durable project-wide decisions once in `CLAUDE.md` or `.specify/constitution.md`: language choices, testing policy, dependency rules, accessibility standards, error format conventions. Individual feature specs reference the constitution rather than re-litigating these decisions on every spec. Changes to the constitution affect all future specs automatically.

2. **Plans are volatile; specs are permanent** — After implementation completes, discard `plan.md` and `tasks.md`. Keep `spec.md` forever as the record of intent. A plan is a work breakdown; a spec is a contract. Conflating them causes confusion in future sessions.

3. **Bidirectional maintenance** — When an agent discovers that an existing implementation differs from the spec, it should update the spec and report the deviation explicitly: "Found an existing pagination helper — reused it. Updated spec to reference `src/shared/paginate.ts`." A spec that only humans maintain will drift.

4. **Cite specs in commits** — `feat(auth): magic link flow, refs specs/004`. This creates an auditable link between a specification and the code that implements it. Future agents and developers can trace intent from commit back to spec.

5. **Match methodology to task complexity** — SDD is worth the overhead for: migrations, large refactors, unclear or contested requirements, new library adoption, anything that crosses multiple team boundaries. It is not worth it for: simple bug fixes, one-file changes, obvious CRUD additions with no edge cases.

6. **Start each session by reviewing the spec** — Before continuing implementation across sessions, read the current spec and check whether the codebase still matches it. Agents that jump into code without this check are building on a potentially stale foundation.
