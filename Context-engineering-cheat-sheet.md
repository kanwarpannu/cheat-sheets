# Context Engineering Cheat Sheet

## Why Context Engineering Matters

Large contexts degrade agent performance for a few key reasons:
- Hallucinations accumulate over time and get reinforced
- Conflicting logic between earlier and later parts of the context causes inconsistent behavior
- Too much context causes agents to sidetrack or be negatively influenced away from their training

After 30+ tools, agent performance starts degrading. Around 100 tools it typically collapses. Implementing RAG over tool selection can produce significantly better results.

---

## Setting Up a New Codebase

1. Initialize a `CLAUDE.md` file at the root level — this file is always loaded into context. The command is `/init`.
2. This file should describe coding style, project architecture, or point to other files with more detail. Keep it as small as possible since it loads on every session. Try to stay under 175–200 lines.
3. Every internal folder (e.g. a Maven multi-module project) can have its own `CLAUDE.md` file, which is loaded into context once the coding agent enters that folder.
4. `CLAUDE.md` files can be checked into git. If you want something local-only, create a `CLAUDE.local.md` file and add it to `.gitignore`.
5. For project-level permissions, add them to `.claude/settings.json` and check it into git. For local-only permissions, use `.claude/settings.local.json` (no need to check in).
6. For project-level MCP servers, add them to `.mcp.json`.

---

## Working with Large Codebases

Repos that are several gigabytes in size, or files with 50,000+ lines, break naive "read the whole file" approaches. The goal is to load only the slice of context relevant to the task, not the whole codebase. Here are six strategies that work today:

1. **Layered CLAUDE.md files** — create a root `CLAUDE.md` (≤200 lines) plus a separate `CLAUDE.md` inside each major module or service directory. Each module file should name its key classes, their approximate sizes, coding patterns, and common gotchas. The agent automatically loads only the file for the folder it's working in, keeping context tight.

2. **Deny rules in `.claude/settings.json`** — block build artifacts, compiled bytecode, and generated code from ever being read. Add entries like `"Read(./**/target/**)"`, `"Read(./**/*.class)"`, and `"Read(./**/*.jar)"` to the `permissions.deny` list and check this file into git so every developer benefits.

3. **Path-scoped rules in `.claude/rules/`** — instead of cramming everything into CLAUDE.md, create separate markdown files in `.claude/rules/` with a `paths:` frontmatter. They auto-apply only when the agent touches matching files (e.g. apply security constraints only to `**/*Service.java`), keeping each rule file focused.

4. **Use subagents for exploration** — when understanding a subsystem requires reading many files, delegate it to a subagent. Say: *"Use a subagent to find all classes involved in the payment flow and summarize how they connect."* The subagent works in its own context window; your main session stays clean for implementation.

5. **Navigate with grep/find, not whole-file reads** — for large files, instruct the agent (via CLAUDE.md) to locate methods with `grep` or `find` before reading. Also list your largest files by name in the root CLAUDE.md so the agent knows upfront not to read them whole.

6. **Git worktrees with sparse checkouts** — when running multiple agents on the same repo simultaneously, use worktrees and set `sparsePaths` in settings so each agent only checks out the module directory it needs, reducing disk I/O and context noise.

7. **Knowledge graph MCP server** — parse your entire Java codebase into a structural graph (zero LLM cost, pure tree-sitter AST) and connect it to your AI agent as an MCP server. The agent queries "what calls this method?" or "what's the blast radius of changing this class?" via graph tools instead of reading files. One query returns ~235–350 tokens vs ~61k for grep+read. Complements the six strategies above — see the Useful MCP Servers section for Graphify and codebase-memory-mcp setup.

### Layering CLAUDE.md with a Knowledge Graph

CLAUDE.md files and knowledge graph MCP servers solve different halves of the large-codebase problem and are designed to be used together:

| Layer | What it holds | When it loads |
|---|---|---|
| **CLAUDE.md** (per module) | Module purpose, key class names, coding conventions, "don't read these files whole" hints | Always — every session, automatically |
| **Knowledge graph MCP** | Structural graph: call chains, inheritance, blast radius, cross-module dependencies | On demand — queried only when the agent needs to navigate |

In practice: keep CLAUDE.md files focused on orientation (≤200 lines per module). Add one pointer line such as `"For cross-module structure, query the knowledge graph via graphify_overview or search_graph."` The graph handles structural navigation; CLAUDE.md handles static orientation.

For Java monoliths specifically: list your top 10–15 most-connected classes and their module in the root CLAUDE.md. The knowledge graph then handles the call-chain depth the CLAUDE.md can't afford to document.

### AST Parsing and Knowledge Graph Tools

AST (Abstract Syntax Tree) parsing converts source code into a structural tree of classes, methods, signatures, and call relationships. Instead of grepping a 50K-line file, an AST-aware tool can answer "find all callers of `getUserById`" with full type precision — no false positives from comments or strings.

This is a complementary technique to the strategies above, not a replacement. The underlying tech stack:
- **tree-sitter** — universal parser for 158+ languages including Java; production-proven, used in GitHub and Neovim. The foundation of both Graphify and codebase-memory-mcp.
- **Hybrid LSP** (codebase-memory-mcp) — adds type-aware resolution on top of tree-sitter for Java: resolves generics, lambdas bound to functional interfaces, overloaded methods, and annotation-bound types that pure AST misses.
- **LSP + jdt** — Eclipse's Java language server; mature, IDE-level resolution if you want to wire up structure-aware navigation without an MCP server.

Both Graphify and codebase-memory-mcp are production-ready today and require no custom wiring — install, point at your repo, and the agent gets graph tools automatically. For Java monoliths: **codebase-memory-mcp's Hybrid LSP** gives the most precise Java type resolution. **Graphify** is the better choice if you also need to ingest architecture docs, SQL schemas, or PDFs alongside the code.

---

## Effective Prompting Strategies

1. Start by saying: *"I want to build this new feature — can you explore the codebase and suggest 2–3 different ways to build it before making any changes?"*
2. Select an approach, then say: *"Create a plan before you start editing files."*
3. End every prompt with: *"Ask me as many questions as needed until you are 95% confident you understand exactly what I need. Don't make assumptions."*
4. Ask it to show the full plan, add your own checkpoints at each step, and tell it not to move on until you are 95% confident the previous task is complete.

---

## Common Workflows

1. **Explore → Plan → Confirm → Code → Commit** — used for most common coding tasks
2. **Write Tests → Commit → Code → Iterate → Commit** — TDD approach, known to be great for backend development
3. **Write Code → Screenshot Result → Iterate** — for frontend coding; give the agent a screenshot tool to get great visual feedback

---

## Managing Context Day-to-Day

1. Use different git worktrees when running multiple coding agents on the same repo simultaneously.
2. Ask the coding agent to suggest commit messages — it's great at it.
3. In Claude Code, run `/context` to inspect what's currently in context if it feels bloated. Then use `/compact` and tell it what to remove. Do this around the time context hits ~60% full.
4. Ask the agent to do things in parallel by saying: *"Use subagents to do these 3 things in parallel: 1. item1, 2. item2, 3. item3 — use the Haiku model for it."*
5. Use `/rewind` to go back one step in the chat.

---

## Skills and Customization

To create a custom skill:
1. Inside `.claude/`, create a folder named `skills`.
2. Inside `skills/`, create a subfolder for each skill — the folder name becomes the skill name.
3. Inside each skill folder, create a `SKILL.md` file that describes the skill to Claude.

---

## Useful MCP Servers

1. **Context7** — pulls version-specific documentation and code examples for libraries directly into the agent's context window, preventing hallucinated or outdated API usage.
2. **Ponytail** — injects a "lazy senior developer" ruleset that enforces smaller diffs and fewer dependencies, pushing agents toward thoughtful, minimal solutions rather than sprawling code.
3. **Superpowers** — a library of 14 structured workflow skills (including TDD, systematic debugging, writing/executing plans, parallel agents, git worktrees, and code review) that enforce disciplined step-by-step processes — essentially a formalized, automated version of the Common Workflows above. Invoke any skill naturally: *"Use the test-driven-development skill to implement this feature"* or *"Recommend skills for building this."*

4. **Graphify** — turns your codebase into a queryable knowledge graph your AI assistant navigates instead of reading raw files. No embeddings, no vector store — pure structural graph built from AST parsing. The assistant queries the graph via MCP tools, getting structural maps instead of walls of code. ([Graphify](https://github.com/Graphify-Labs/graphify) · [graphify-mcp](https://github.com/yasinyaman/graphify-mcp))

   - **Graph build — zero token cost for code.** Uses tree-sitter AST parsing: no LLM calls, no API cost. Run `graphify extract ./src --code-only` to parse Java files into nodes (classes, methods, fields) and edges (calls, inheritance, imports). Outputs `graph.json`, an interactive `graph.html` visualization, and a `GRAPH_REPORT.md`. For large monoliths, `--max-workers 16` parallelizes across modules. Only non-code assets (docs, PDFs) cost tokens — skippable.
   - **MCP server.** Run `python -m graphify.serve graphify-out/graph.json` (stdio, local dev) or add `--transport http --host 0.0.0.0` for a team-shared instance. The assistant connects and queries through MCP tools — no files are read directly.
   - **Key MCP tools.** Call `graphify_overview` first (god nodes, communities, anomalies). Then: `graphify_subgraph` (token-budgeted k-hop neighborhood around a class), `graphify_impact` (reverse-dependency blast radius — "what breaks if I change X?"), `graphify_skeleton` (all method signatures with annotations, bodies stripped — great for large Java classes), `graphify_locate` (NL query → graph node + `hidden_links` for hidden duplication), `graphify_path` (shortest dependency path between two classes), `graphify_cycles` (circular dependency detection), `graphify_freshness` (is the graph stale?), `graphify_build(update=True)` (incremental rebuild).
   - **Token efficiency.** One `graphify_locate` call costs ~235 tokens vs ~61k for grep+read — 263× fewer. Tools return structural maps; code is read surgically via `graphify_skeleton` / `graphify_fetch` only when needed.
   - **Git freshness.** `graphify_freshness` returns `fresh / update / rebuild` + reason. Wire with `graphify hook install` to auto-rebuild on commits. Cosmetic-only changes (reformats, comments) never trigger needless rebuilds — AST-diffed.
   - **Java & enterprise monoliths.** Install `pip install "graphify-mcp[treesitter]"` for Java support. Resolves cross-module method calls, inheritance chains, and package imports automatically. `graphify_cycles` and `graphify_impact` are the most valuable tools for large monoliths.

5. **codebase-memory-mcp** — a single static binary (zero runtime dependencies) that indexes your codebase into a persistent knowledge graph and exposes it as 15 MCP tools. One-line install auto-configures 43 clients (Claude Code, Cursor, VS Code, etc.). Strong alternative to Graphify for pure-code graphs; complements it if you also need non-code ingestion. ([GitHub](https://github.com/DeusData/codebase-memory-mcp))

   - **Indexing — zero dependencies.** Precompiled binary; no Python, no Node, no package manager needed. Indexes typical repos in milliseconds; the Linux kernel (28M LOC, 75K files) in ~3 minutes. Graph stored in SQLite at `~/.cache/codebase-memory-mcp/`. One-line install: `curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash`
   - **Java Hybrid LSP.** Goes beyond tree-sitter: resolves generics, lambdas bound to functional interfaces, overloaded methods, annotation-bound types, and class hierarchies with dispatch resolution. The strongest Java type resolution of any open-source MCP graph tool.
   - **Key MCP tools.** `get_architecture` (overview — call first), `search_graph` (regex + label filters), `trace_path` (call chains between two symbols), `detect_changes` (git diff blast radius), `get_code_snippet` (fetch actual source for a node), `search_code` (grep-augmented), `query_graph` (read-only openCypher for power users).
   - **Token efficiency.** 99.2% token reduction on structural queries — ~3,400 tokens for five queries vs ~412,000 via grep. Results are tightly focused (~100–350 tokens per query vs ~1,500 for Graphify BFS).
   - **vs. Graphify.** codebase-memory-mcp wins on install simplicity, Java type precision, and focused query results. Graphify wins on multi-modal ingestion (docs, PDFs, SQL schemas) and interactive visualization (`graph.html`). For a pure-code graph on a Java monolith, codebase-memory-mcp is typically the simpler starting point.

---

## Cursor AI Equivalents

This section maps each Claude Code concept to its Cursor equivalent. Cursor is an IDE extension rather than a full agent framework, so some features have no direct parallel.

### Setting Up a Codebase

- **`CLAUDE.md`** → create a `.cursorrules` file at the project root. Same purpose — coding style, architecture conventions, agent guidance. No `/init` command; create it manually.
- **Per-folder `CLAUDE.md` files** → not supported. `.cursorrules` is root-only. Use `.cursor/rules/*.mdc` files with a `globs:` field to scope rules to file patterns instead (e.g. `**/*.java` to target all Java files).
- **`.claude/rules/` path-scoped rules** → `.cursor/rules/*.mdc` files with YAML frontmatter. Fields: `description` (one-liner summary), `globs` (file patterns that auto-attach the rule), `alwaysApply: true/false` (true = applies to every request, false = only when matching files are in scope).
- **`.claude/settings.json` deny rules** → no equivalent. Cursor has no mechanism to block specific file reads.
- **`.mcp.json`** → `.cursor/mcp.json` at the project root for project-level MCP servers. Global MCP servers are configured in Cursor's user settings.

### Working with Large Codebases

- **Layered CLAUDE.md strategy** → use `.cursor/rules/*.mdc` files with `globs:` to approximate module-level scoping by file pattern (e.g. one `.mdc` for `src/api/**`, another for `src/data/**`).
- **Deny rules** → no equivalent in Cursor.
- **grep/find navigation** → identical advice applies; works in Cursor's terminal and Agent mode.
- **Subagents for exploration** → no equivalent. Cursor operates in a single context window.
- **Knowledge graph MCP server (strategy 7)** → works identically in Cursor. Add Graphify or codebase-memory-mcp to `.cursor/mcp.json`. The same layering principle applies: use `.cursorrules` or `.cursor/rules/*.mdc` files for static orientation, and the knowledge graph MCP for on-demand structural queries.

### Managing Context Day-to-Day

- **`/context`, `/compact`, `/rewind`** → no equivalents. Cursor manages the context window automatically and does not expose these controls.
- **Parallel agents + git worktrees** → no equivalent.
- **Commit message suggestions** → works the same; just ask Cursor to suggest a commit message.

### Common Workflows

All three workflows (Explore → Plan → Confirm → Code → Commit, TDD, Screenshot → Iterate) work in Cursor with the same prompting discipline. Screenshot iteration is especially smooth — paste images directly into Cursor chat without any extra tooling. There is no built-in workflow enforcement; document your preferred workflow in `.cursorrules` to guide the agent's behavior.

### Skills and Customization

No equivalent for the `.claude/skills/` system. Document reusable workflows as named sections in `.cursorrules` instead.

### MCP Servers

Context7, Ponytail, Superpowers, Graphify, and codebase-memory-mcp all work with Cursor. Configure them in `.cursor/mcp.json` at the project root.
