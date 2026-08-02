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

### AST Parsing and Language Servers (Watch This Space)

AST (Abstract Syntax Tree) parsing converts source code into a structural tree of classes, methods, signatures, and call relationships. Instead of grepping a 50K-line file, an AST-aware tool can answer "find all callers of `getUserById`" with full type precision — no false positives from comments or strings.

This is a complementary technique to the strategies above, not a replacement. Key tools to know:
- **tree-sitter** — universal parser for 100+ languages; production-proven, used in GitHub and Neovim
- **LSP (Language Server Protocol)** — IDE-agnostic standard for structure-aware code intelligence (definitions, references, symbols)
- **jdt** — Java-specific language server (part of Eclipse); mature and well-suited to large Java monoliths

Using AST/LSP specifically as a *context management strategy for AI agents* is still emerging — production tools like Copilot and JetBrains AI use some form of it internally, but there's no clean plug-and-play open-source solution yet. For Java monoliths, **LSP + jdt is the most practical path** if you want to experiment with structure-aware navigation today.

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

---

## TODO

- The above stuff needs to have Cursor AI equivalents as well. ????
