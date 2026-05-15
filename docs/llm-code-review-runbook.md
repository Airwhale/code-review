# LLM code-review runbook

Operational guide for using `review.py` as an iteration partner during code work. Same workflow whether you're a human developer or a coding agent (Claude, Codex, etc.).

The runner is a thin Python wrapper around the upstream `gemini-cli-extensions/code-review` skill + command prompts. It POSTs them to either OpenRouter or the Gemini API and prints structured-markdown findings — same model, same prompts, same output shape as the GitHub `/gemini review` bot, without the 5–15 min webhook → job-queue wait.

---

## Setup

```
Repository:    https://github.com/Airwhale/code-review
Entry point:   review.py at the repo root
Dependencies:  uv-managed -- first run installs them
Config:        .env at the runner's repo root (NOT at the project being reviewed)
```

Set the key for whichever provider you'll use; missing keys fail fast with a clear error:

- `OPENROUTER_API_KEY` — default provider
- `GEMINI_API_KEY` — direct Google AI Studio path

Optional: `CODE_REVIEW_PROVIDER`, `OPENROUTER_MODEL`, `GEMINI_MODEL` override the defaults.

---

## Invocation

From any project directory:

```bash
cd /path/to/your-project
uv run --project /path/to/code-review /path/to/code-review/review.py --base origin/main
```

Or from the runner directory against an external CWD:

```bash
cd /path/to/code-review
uv run review.py --pr 42
```

The runner reads `.env` from its own directory — configure once, invoke from anywhere.

### Diff modes (mutually exclusive)

| Flag                 | Diff scope                                      | Use when                                |
|----------------------|-------------------------------------------------|-----------------------------------------|
| *(none)*             | merge-base against `origin/HEAD`                | quick check on current branch           |
| `--base origin/main` | two-dot diff vs ref, **includes working tree**  | iterating before commit                 |
| `--pr <N>`           | `gh pr diff N` (requires `gh auth login`)       | reviewing an existing PR                |
| `--staged`           | staged-only                                     | pre-commit hook style                   |

### Providers

| `--provider`    | Env key required        | Default model            | Notes                                                                                   |
|-----------------|-------------------------|--------------------------|-----------------------------------------------------------------------------------------|
| `openrouter` *  | `OPENROUTER_API_KEY`    | `google/gemini-2.5-pro`  | Default. Reliable quota; recommended for iterative work.                                |
| `gemini`        | `GEMINI_API_KEY`        | `gemini-2.5-pro`         | Direct to Google AI Studio. **Free tier has zero quota for pro** — use flash if free.   |

\* default

`--model <slug>` overrides the default. The `flash` variant is ~3× faster than `pro` with some quality loss — use during heavy iteration, switch to `pro` for the final pass.

---

## Iteration loop

```
1. Edit the target repo (fix a bug, build a feature, etc.).
2. Run:  uv run --project <runner> <runner>/review.py --base origin/main
3. Read the structured-markdown output. Findings are tagged
   CRITICAL > HIGH > MEDIUM > LOW.
4. For each finding, decide: accept or decline.
5. Apply accepted fixes inline. Do NOT commit yet.
6. Re-run step 2.
7. Repeat until output is:
      "No issues found. Code looks clean and ready to merge."
8. Run tests + build. Commit. Push.
```

**Do not commit between rounds.** `--base origin/main` uses a two-dot diff (`git diff -U5 <base>`), which includes working-tree edits. Re-running picks up in-progress fixes immediately. Committing every round produces noisy history; the reviewer is happy reviewing uncommitted edits.

---

## Accept / decline heuristics

**Accept by default:**

- **CRITICAL / HIGH** unless the finding is factually wrong (rare). Real correctness or security bugs.
- **MEDIUM** about correctness, concurrency, atomicity, latent bugs, schema or type safety.
- **MEDIUM** about defensive coding — wider exception catches, header normalization, partial-key cache invalidation. Cheap hardening.
- **LOW** when trivially right: consolidating duplicates, fixing typos, tightening assertions, correcting inaccurate comments.

**Decline (and add a code comment explaining why):**

- Findings that contradict load-bearing design intent already encoded. *Shape:* a pair of API endpoints that look redundant but are deliberately split for a planned future migration. The reviewer doesn't know your roadmap unless your code comments tell it.
- Findings that would untighten a deliberately-tight test. *Shape:* a hardcoded expected count the reviewer wants computed dynamically from the fixture — that turns the test into a tautology (*what the code reads == what the code reports*) and removes the regression-catch.
- Stylistic preferences that don't change correctness when the existing form has a defensible rationale. *Shape:* a cache `maxsize` chosen with deliberate headroom for test fixtures rather than the obvious-singleton value.

---

## The decline contract

**Every declined finding gets a code comment immediately adjacent to the flagged code**, explaining why the suggestion was rejected. Without the comment, the next iteration's model will surface the same finding again. The comment is a contract with future review rounds — it's how you teach the reviewer about decisions it cannot infer from the diff alone.

This is the central operational rule. Without it, the loop churns on the same findings indefinitely. With it, the loop converges to clean in a small number of rounds.

---

## Known gotchas

1. **Transient `None` output.** OpenRouter occasionally returns an empty completion (content filter, provider hiccup, rate-limit interstitial). The runner surfaces this as a literal `None`. Just rerun the same command — the second call has always worked in practice. Don't debug it.

2. **Free-tier 429 on Gemini direct.** `--provider gemini --model gemini-2.5-pro` requires a paid Google AI Studio plan; the free tier returns HTTP 429 immediately. Either use `--provider openrouter` (preferred) or `--model gemini-2.5-flash`.

3. **Two-dot vs three-dot diff.** `--base <ref>` uses two-dot (`git diff -U5 <ref>`) so working-tree changes show up. Three-dot (`<ref>...HEAD`) would show only committed changes and the reviewer would keep re-flagging the same issues. The two-dot semantics is intentional for the iteration workflow.

4. **Diff size shapes round count.** Larger diffs surface more findings per round and take more rounds to converge. Rough observed shapes:
   - Small PR (~25K-char diff, single feature): 3–4 rounds
   - Medium PR (~50K chars): 4–6 rounds
   - Large PR (~300K chars): 8–12 rounds

5. **`tee` to a file.** When output is large, pipe to `tee /tmp/review.md` so you can re-read findings without re-invoking the tool. Saves context budget on subsequent steps.

---

## When to also call `/gemini review` on GitHub

Treat the local tool as the **iteration partner** and the GitHub `/gemini review` bot as the **final-mile verifier**. They use the same model and similar prompts; the GitHub bot's only advantage is independence ("a third party reviewed this, not my own prompt-following loop").

Bring in the GitHub bot when:

- The diff touches concurrency, locks, signing/replay, differential privacy ledgers, policy boundaries, or auth surfaces — anywhere a missed bug has outsized blast radius.
- The PR is large enough that you want a credibility signal beyond "I ran the same model locally."
- You're about to merge to a protected branch and want one independent confirmation.

For most PRs, three to five clean local rounds is sufficient and saves 30–45 minutes per PR vs the webhook round-trip.

---

## Per-round tracking

Keep a one-line ledger per round so the final commit message or PR comment can summarize the cycle:

```
Iter N: <severity> <one-line-description> -> applied | declined-with-comment | clean
```

After the cycle, the ledger becomes a table in the commit body or PR comment:

```
| Round | Finding                                            | Action                |
|-------|----------------------------------------------------|-----------------------|
| 1     | MED: cache duplicated across modules               | Applied               |
| 1     | LOW: lru_cache maxsize stylistic nit               | Declined w/ comment   |
| 2     | MED: CWD-relative path resolution is fragile       | Applied               |
| 3     | None -- "No issues found. Code looks clean..."     | Ready                 |
```

This is the artifact a human reviewer reads to understand what changed and why. Don't skip it.

---

## One-liner reference

The command shape used most often during iterative work:

```bash
uv run --project /path/to/code-review /path/to/code-review/review.py --base origin/main 2>&1 | tee /tmp/review.md
```

Tail the file in another shell, or pipe to `head -80` if you only want the top findings.

---

## Provenance

This fork keeps the upstream `gemini-cli-extensions/code-review` skill and command prompts byte-identical so upstream improvements rebase cleanly. Fork additions: `review.py`, `pyproject.toml`, `.env.example`, `.gitignore`, this runbook, and a rewritten root `README.md`. See the root README for the full list of fork modifications and how to sync against upstream.
