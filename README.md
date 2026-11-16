# make-no-mistake

> "Make no mistake" is not a constraint. It is a wish.

`make-no-mistake` is an agent skill that turns the old vibe-coding incantation into an actual verification loop. It does not promise perfection. It makes the work checkable: define correctness, hunt for specific failure modes, demand observations instead of self-confidence, fix only what is proven broken, and report what remains unknown.

## The idea

Most "be careful" prompts add no information. They do not say what a mistake is, which mistakes matter, or how anyone would know. This skill supplies the missing machinery:

- **Contract** — the goal, observable done criteria, non-goals, assumptions, and stakes (with reversibility checked across technical, data, information, and operational dimensions).
- **Mistake map** — a pre-mortem across wrong goal, wrong path, wrong facts, wrong change, wrong evidence, and wrong state.
- **Predict, then act** — write the expected observation before acting; a surprise is a signal.
- **Verify** — establish provenance and oracle independence, pick the domain-appropriate verification mechanism (software, finance, legal, research, documents, operations), rank evidence, test the test, inspect the diff, check the boundary.
- **Correct** — confirmed defects only, one fix at a time, with a regression check.
- **Fresh eyes** — a separate fresh-context verifier (isolated subagent, clean CLI model process, or human) with least-privilege access; for T1 only, an honest solo cold-read with executed checks kept `verified` and unexecuted behavioral claims marked `inferred`.
- **T2 gate** — a pre-execution packet covering blast radius and four-dimensional reversibility, pre-state, rollback, abort criteria, post-action checks, approval, and harness-level hardening.
- **Report** — tier triggers, verified, inferred, unverified, residual risk, verifier, stop reason, and (for T2) ledger, post-action state, and rollback.

It scales ceremony to stakes: **T0** uses five questions at the top of `SKILL.md`, **T1** runs the compact core loop, and **T2** loads [`references/t2.md`](references/t2.md) for a persisted risk ledger, a mandatory fresh-context verifier, and — whenever an action crosses the gate boundary — an explicit gate packet and post-action verification.

## Why it is not just "try harder"

A model asked to check its own work tends to agree with itself. This skill makes verification external and falsifiable: state the claim, name the observation, make the observation, and distinguish what was verified from what was inferred or assumed. It also has a stopping rule, because endless checking is another failure mode.

## Install

The skill package is `SKILL.md` plus four reference files loaded on demand, with no runtime dependencies. Preserve the `make-no-mistake/references/` directory layout when installing or uploading so relative links resolve:

```text
make-no-mistake/
├── SKILL.md
└── references/
    ├── evidence.md
    ├── t2.md
    ├── verifier.md
    └── worked-example.md
```

```bash
mkdir -p <skills-dir>/make-no-mistake
cp -r SKILL.md references <skills-dir>/make-no-mistake/
```

## Hardening the gate (operator setup)

A gate written in skill text is soft policy: a miscalibrated shell, interpreter (`python`, `node`, `curl`), or MCP tool call can still cross the gate boundary unless the runtime blocks it. Keep live production credentials unmounted during pre-gate work and enforce the boundary in your harness:

- **Codex:** `--sandbox` (`read-only` or `workspace-write`), `--ask-for-approval` (`approval_policy`), and `.rules` execpolicy rules.
- **Claude Code:** `permissions` (`allow` / `ask` / `deny` in `.claude/settings.json`) and `PreToolUse` hooks covering `Bash` and MCP write/high-impact-read tools.
- **Cursor:** command and MCP allowlists (`permissions.allow` / `permissions.deny` in `.cursor/cli.json` or IDE settings), `sandbox` (`enabled` with `networkAllowlist`), and project or enterprise `.cursor/hooks.json` (`beforeShellExecution`, `beforeMCPExecution`, `preToolUse` with `failClosed: true`). Note that user-level `~/.cursor/hooks.json` does not reach Cloud Agents, and CLI hooks can fail to fire even with `failClosed: true`.

**Smoke-test the gate before relying on it:** trigger a harmless forbidden sentinel command in a throwaway sandbox and confirm it is actually denied before trusting hooks or allowlists around production. *(Checked 2026-11-17 against `codex-cli 0.167`, `Claude Code 2.1`, and `Cursor CLI 2026.11`; flags and hook behavior change — verify against current vendor documentation and `<cli> --help`.)*

## Use

See the `description` frontmatter in [`SKILL.md`](SKILL.md) for the authoritative trigger rules, or invoke `/make-no-mistake` explicitly when your harness supports slash-skill invocation.

Examples:

- "Add rate limiting to the login endpoint. Make no mistake."
- "Here is my migration plan. Double-check the order before I run it."
- "The test passed but the behavior is still wrong. How do you know the fix works?"

## Honest limits

- It cannot guarantee zero mistakes. It replaces "trust me" with "here is the evidence and the residual risk."
- It costs time and tokens. Use it where being wrong is expensive, not as ceremony on every edit.
- A separate verifier helps, but the same model can share the same blind spots. Agreement is not verification.
- It does not replace tests, type checks, code review, or human judgment.
- Prompt rules alone are soft policy: T2 verifier requirements, waivers, and execution gates are defined in [`references/t2.md`](references/t2.md); back them with the smoke-tested harness controls above.

## The name

"Make no mistake" became a joke about magical prompting: a huge, underspecified request plus an imaginary `--no-bugs` flag. `thesysdev/make-no-mistakes` is the satirical M-Stack version. This project is the sincere inverse: not a magic flag, a loop.

The name is a joke. The loop is not.
