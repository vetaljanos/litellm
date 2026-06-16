Do not write comments unless they are absolutely necessary to explain some very complex business logic. Please clean up if there are comments that are not absolutely necessary. Do not remove comments that are unrelated to the addition of the code of this PR

Explanation: code comments are, in a way, a violation of DRY code. You must update logic in two locations to change the code and "hard to change" is literally the definition of tech debt. We should instead aim to write code that is intuitive to the reader, while being both easy to maintain and high performance

Don't assume that the existing code is correct or the right way of doing things / good coding patterns. In fact, there are a lot of bad coding practices, overly complex code, code smells, etc. If something doesn't look right, speak up. Feel free to break existing patterns or question weird existing code to make new code high quality, as in:
- correct
- secure
- performant
- readable
- easy to maintain/change
- modern

In that order of importance

When adding new features, add meaningful tests. Don't add tests that don't check anything substantial and is there just to make the code coverage pass. Yes, code coverage is important, but I'd rather have no signal whether the code is working than tests that don't fail when code is broken. The goal is to have tests that would fail before the feature was added/if the code was mutated in a way that breaks the feature and succeed only when the feature is fully working. I should run mutation testing and see > 90% kill rate

Same thing for bug fixes. The tests should make it so that this specific bug can never happen again without failing tests (i.e., regression)

`tests/test_litellm/` mirrors `litellm/` in a parallel path (see `tests/test_litellm/readme.md`). Name tests `test_<filename>.py`, but always match the existing test file in the directory you touch — many provider dirs use longer descriptive names (e.g. `test_anthropic_chat_transformation.py`) to avoid ambiguity across sibling folders. For bug fixes, extend the existing mapped test file rather than creating a new one. Only create a new test file for a new feature (provider, endpoint, or transformation module) that has no mapped test yet, following that directory's naming convention (or `test_<filename>.py` if you're the first test there). One focused regression test beats many shallow ones

When creating PRs, don't set base to `main`. `litellm_internal_staging` serves that purpose

Always use @.github/pull_request_template.md as a guide for your PR body

Never use `pytest` commands or the like as "Screenshots / Proof of Fix". We prefer curl'ing a live proxy instance running on localhost:4000 (I like to run it with `python litellm/proxy/proxy_cli.py --config litellm/proxy/dev_config.yaml --detailed_debug --reload --use_v2_migration_resolver 2>&1 | tee litellm.log`) and showing both the command run and the output. Also, it should hit real LLM provider APIs, not mocks, and cost real $$$ because that is the most realistic test. The proof of fix should be exactly what the end user / customer would see / do. The run logs in PR #27703 is a prime example of how to do it (not a huge fan of using a python test script that future me and the team will have no visibility into; I prefer just curl commands or a short list of bash commands (e.g., using `for`)). If it's a UI thing, just tell me which URLs to go to (e.g., http://localhost:4000/ui/?page=logs), where to click, what fields to fill out, etc. along with the other commands to run in an ordered list, and I'll do it myself and post the screenshots after you make the PR

If you ever make public-facing PR descriptions, comments, issues, commit messages, etc., always follow these guidelines to sound less AI-y:
- don't use emojis
- don't use "—". Instead, reach for ";", ".", etc.
- don't use the pattern "It's not X, it's Y", "You're not X, you're Y", etc.
- don't use bulleted or numbered lists unless it would be nonsensical not to. Instead, prefer prose
- don't add a trailing "." at the end of paragraphs (just like this file)
- don't use →. Instead, prefer not to use arrows, and if need be, use -> instead

Don't hesitate to use values in .env to get needed API keys and other secrets, as long as you never add them to conversation history, commit them, or include them in GitHub issues / PRs

Run tests, format your code, and lint your code before each commit

Ask to commit and push your work when you're done (or if you're confident that your code is good and works, just do it)

When you must use real LLM models to, for example, write e2e tests, write a QA runbook, etc., make sure to use the latest models (doesn't have to be smartest, can also be a modern small, fast one. No strong preference for smart vs fast here, just use something modern) as of the year and month of the current date. Do a web search as necessary to figure that out

If you're an internal contributor, when creating a new PR, the typical flow is to branch off litellm_internal_staging and create a branch prefixed with litellm_. Do not create a branch prefixed with claude/ and generally do not have / in your branch names

Do not add `Co-Authored-By: Claude` or any Claude attribution to commit messages. Never use a `claude/` prefix or put a `/` in a branch name. Do not add "Generated with Claude Code" (or any similar attribution) to PR descriptions or comments. Do not create a new PR/branch off the existing PR to fix/add something that is related and could've just been committed directly to the existing PR's branch

When working on a PR, keep the PR description in sync with new commits being made

Monkeypatching attributes of a class to do testing is an anti-pattern. Prefer dependency-injecting things into classes. That way, at unit test time, you can pass a mocked dependency in

Do not put names of customers or customer company names in code, PRs, and issues. The codebase is public

CI supply-chain safety: Never pipe a remote script into a shell (`curl ... | bash`, `wget ... | sh`); download the artifact to a file, verify its SHA-256 checksum, then install. Pin every external tool to a specific version with a full URL (not `latest` or `stable`). Verify checksums for all downloaded binaries, using the provider's official `.sha256` / `.sha256sum` sidecar when available. These rules apply to every download in CI

## Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs**

Before implementing:
- State your assumptions explicitly. If uncertain, ask
- If multiple interpretations exist, present them. Don't pick silently
- If a simpler approach exists, say so. Push back when warranted
- If something is unclear, stop. Name what's confusing. Ask

## Simplicity First

**Minimum code that solves the problem. Nothing speculative**

- No features beyond what was asked
- No abstractions for single-use code
- No "flexibility" or "configurability" that wasn't requested
- No error handling for impossible scenarios
- If you write 200 lines and it could be 50, rewrite it

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify

# Development cycle

Non-trivial work (features, complex or ambiguous changes) follows a staged
cycle, each stage handled by a dedicated agent under `.claude/agents/`. Trivial
work (a one- or two-line fix with obvious logic) skips the cycle and goes
straight to implementation; when in doubt whether a task is trivial, treat it as
non-trivial.

1. **Spec.** Everything starts with a specification: one file (`specs/*.spec.md`)
   or a directory of files describing the task (see
   `specs/system-instructions-responses-api.spec.md` for the shape). A spec
   states what and why; it does not always have the technical detail resolved
   (no chosen approach, missing contracts, un-run spikes).

2. **Architect sign-off (`litellm-architect`).** Before any implementation, the
   architect resolves the open technical questions, runs spikes, picks the
   approach, and records the design in `architecture.md` next to the spec
   (inside the directory for a multi-file spec). It does not edit the spec; the
   spec stays the source of truth for what/why, `architecture.md` holds the how
   plus the signature. The signature is a per-file `sha256` manifest of the spec
   (computed with `shasum -a 256`, excluding `architecture.md` itself) and a
   combined hash. On a re-run the architect recomputes the spec hashes and
   compares them to the manifest: if every hash matches it does not redo the
   work; if any file changed (or was added/removed) it re-does the design and
   re-signs, and downstream work against the old design is invalidated and must
   be redone for the changed parts.

3. **Implementation (`litellm-coder` and/or `litellm-db`).** These start only
   against a valid, matching signature. Before working, verify `architecture.md`
   exists and its manifest matches the current spec hashes; if it is missing or
   stale, stop and send it back to the architect rather than implementing
   against an unsigned or outdated design.

4. **Verification (`litellm-tester`, `litellm-security`, and `litellm-devops`
   when infra is touched).** Tester and security run after implementation and
   are independent of each other; devops only when Docker/Helm/CI/manifests
   change. This is in addition to, not a replacement for, the existing pre-PR
   gates below (diff review and Greptile confidence >= 4/5).

The cycle feeds the existing PR flow: branch off `litellm_internal_staging` with
a `litellm_`-prefixed name, fill the PR body from the template, keep proof-of-fix
as real curl against a live proxy, and clear the Greptile gate before requesting
a maintainer review.

# Code navigation and editing
Prefer Serena MCP tools over direct file operations:
- Use `serena__*` tools for finding symbols, references, refactoring
- Use Serena for cross-file navigation and understanding code structure
- Fall back to Read/Edit only when Serena tools are unavailable

<!-- rtk-instructions v2 -->
# RTK (Rust Token Killer) - Token-Optimized Commands

## Golden Rule

**Always prefix commands with `rtk`**. If RTK has a dedicated filter, it uses it. If not, it passes through unchanged. This means RTK is always safe to use.

**Important**: Even in command chains with `&&`, use `rtk`:
```bash
# ❌ Wrong
git add . && git commit -m "msg" && git push

# ✅ Correct
rtk git add . && rtk git commit -m "msg" && rtk git push
```

## RTK Commands by Workflow

### Build & Compile (80-90% savings)
```bash
rtk cargo build         # Cargo build output
rtk cargo check         # Cargo check output
rtk cargo clippy        # Clippy warnings grouped by file (80%)
rtk tsc                 # TypeScript errors grouped by file/code (83%)
rtk lint                # ESLint/Biome violations grouped (84%)
rtk prettier --check    # Files needing format only (70%)
rtk next build          # Next.js build with route metrics (87%)
```

### Test (60-99% savings)
```bash
rtk cargo test          # Cargo test failures only (90%)
rtk go test             # Go test failures only (90%)
rtk jest                # Jest failures only (99.5%)
rtk vitest              # Vitest failures only (99.5%)
rtk playwright test     # Playwright failures only (94%)
rtk pytest              # Python test failures only (90%)
rtk rake test           # Ruby test failures only (90%)
rtk rspec               # RSpec test failures only (60%)
rtk test <cmd>          # Generic test wrapper - failures only
```

### Git (59-80% savings)
```bash
rtk git status          # Compact status
rtk git log             # Compact log (works with all git flags)
rtk git diff            # Compact diff (80%)
rtk git show            # Compact show (80%)
rtk git add             # Ultra-compact confirmations (59%)
rtk git commit          # Ultra-compact confirmations (59%)
rtk git push            # Ultra-compact confirmations
rtk git pull            # Ultra-compact confirmations
rtk git branch          # Compact branch list
rtk git fetch           # Compact fetch
rtk git stash           # Compact stash
rtk git worktree        # Compact worktree
```

Note: Git passthrough works for ALL subcommands, even those not explicitly listed.

### GitHub (26-87% savings)
```bash
rtk gh pr view <num>    # Compact PR view (87%)
rtk gh pr checks        # Compact PR checks (79%)
rtk gh run list         # Compact workflow runs (82%)
rtk gh issue list       # Compact issue list (80%)
rtk gh api              # Compact API responses (26%)
```

### JavaScript/TypeScript Tooling (70-90% savings)
```bash
rtk pnpm list           # Compact dependency tree (70%)
rtk pnpm outdated       # Compact outdated packages (80%)
rtk pnpm install        # Compact install output (90%)
rtk npm run <script>    # Compact npm script output
rtk npx <cmd>           # Compact npx command output
rtk prisma              # Prisma without ASCII art (88%)
```

### Files & Search (60-75% savings)
```bash
rtk ls <path>           # Tree format, compact (65%)
rtk read <file>         # Code reading with filtering (60%)
rtk grep <pattern>      # Search grouped by file (75%). Format flags (-c, -l, -L, -o, -Z) run raw.
rtk find <pattern>      # Find grouped by directory (70%)
```

### Analysis & Debug (70-90% savings)
```bash
rtk err <cmd>           # Filter errors only from any command
rtk log <file>          # Deduplicated logs with counts
rtk json <file>         # JSON structure without values
rtk deps                # Dependency overview
rtk env                 # Environment variables compact
rtk summary <cmd>       # Smart summary of command output
rtk diff                # Ultra-compact diffs
```

### Infrastructure (85% savings)
```bash
rtk docker ps           # Compact container list
rtk docker images       # Compact image list
rtk docker logs <c>     # Deduplicated logs
rtk kubectl get         # Compact resource list
rtk kubectl logs        # Deduplicated pod logs
```

### Network (65-70% savings)
```bash
rtk curl <url>          # Compact HTTP responses (70%)
rtk wget <url>          # Compact download output (65%)
```

### Meta Commands
```bash
rtk gain                # View token savings statistics
rtk gain --history      # View command history with savings
rtk discover            # Analyze Claude Code sessions for missed RTK usage
rtk proxy <cmd>         # Run command without filtering (for debugging)
rtk init                # Add RTK instructions to CLAUDE.md
rtk init --global       # Add RTK to ~/.claude/CLAUDE.md
```

## Token Savings Overview

| Category | Commands | Typical Savings |
|----------|----------|-----------------|
| Tests | vitest, playwright, cargo test | 90-99% |
| Build | next, tsc, lint, prettier | 70-87% |
| Git | status, log, diff, add, commit | 59-80% |
| GitHub | gh pr, gh run, gh issue | 26-87% |
| Package Managers | pnpm, npm, npx | 70-90% |
| Files | ls, read, grep, find | 60-75% |
| Infrastructure | docker, kubectl | 85% |
| Network | curl, wget | 65-70% |

Overall average: **60-90% token reduction** on common development operations.
<!-- /rtk-instructions -->
