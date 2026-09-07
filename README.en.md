# AI PR Guardian — an AI quality gate for pull requests

*[Polish version →](README.md)* · [Changelog](CHANGELOG.md) · [MIT licence](LICENSE)

A Claude Code plugin that reviews a pull request with a "regression guard" subagent, forces
a second agent to argue against its findings, and turns the surviving ones into an exit code.
Blocks the push locally and marks the PR on GitHub.

> ⚠️ **An experiment, not a product.** It was built to test one idea: whether a model
> can guard the classes of bug a script cannot catch. It has never run against a real
> pull request or a self-hosted runner, and it is not maintained. The repository stays
> public because the measurement in `docs/STAN.md` may be useful to someone.
>
> **Authorship:** **11 of its 15 commits were written by Claude Code** — the tool largely
> wrote itself. Measured with `git shortlog -sn main` (the other 4 sit under two git
> identities belonging to the same author).
>
> **In my portfolio — but not as a product.** Owner's decision, 2026-09-07: this project
> is shown as **evidence of directing AI agents** — architecture, permission boundaries,
> mandatory verification and measurement of the outcome. Not as deployed tooling, and not
> as code I typed myself. Both caveats above are stated wherever this entry is used.

---

## Why it exists

The project it was built for had a dense net of automation: 39 script guards, 84 tests,
18 smoke tests and 10 golden files *(measured 2026-09-07; when this tool was built, in
August 2026, the figures were 25 / 75 / 7 / 9)*. It still shipped bugs.

Reviewing its known-issues register showed why — **4 of 13 known bug classes had no automated
protection at all**:

- a test that destroyed the shared development database
- `0` treated as "no value", producing a price of 0
- a build that carried changes out of the working directory
- an invisible element chosen as the Largest Contentful Paint candidate

None of these are things you write a unit test for in advance. They are things you notice
once, and then need to never forget.

## How it works

```
zakres.mjs        →  regression-guard subagent  →  critic  →  brama.mjs
(0 tokens)           (own context,                 (tries to     (0 tokens)
 path filter          Read/Grep/Glob only)          disprove)     severity policy
 mirroring ci.yml)                                                → exit code
```

**Scope selection costs nothing.** `zakres.mjs` decides which paths are in play using the same
rules as `ci.yml`, before any model is invoked. No tokens are spent deciding whether to spend
tokens.

**The guard subagent gets its own context** and read-only tools. It cannot change code — it
only reports, and every finding must carry evidence in `file:line` form.

**The critic is mandatory, not optional.** Its job is to disprove the guard's findings.
Anything it rejects stays in the report with the reason attached, so the next session doesn't
rediscover it.

**The gate costs nothing either.** `brama.mjs` maps findings to an exit code through a policy
in `config/severity.json`. No model call, no ambiguity.

## Two layers of enforcement

| Layer | When | Effect |
|---|---|---|
| `pre-push` hook | Before the push leaves your machine | Blocks it |
| GitHub Actions check | On the pull request | Marks the PR, comments inline |

## Stack

Zero-dependency JavaScript (ESM), TypeScript, Bash, GitHub Actions with a self-hosted runner,
Anthropic API. MIT licensed.

## A measured result

The changelog records a stability measurement of **8/8** after the harness was fixed, and —
in `--powtorz` (repeat) mode — separates genuine instability from variance in finding weight.
That distinction matters: a tool that reports different things on identical input is not a gate,
it is a random number generator with opinions.

## Status

Early. Version 0.4.1, built over a short, intense stretch of work. It does what the description
says, but it has been used on one repository, not many. Treat it as a working prototype of an
idea rather than a finished product.

## Licence

MIT — see [LICENSE](LICENSE).
