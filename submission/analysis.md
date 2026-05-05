# Task 1 — Insecure workflow analysis

This is a per-flaw walkthrough of `.github/workflows/ai-review-insecure.yml`. Numbering matches the in-file `# Flaw N` comments. Each entry is one paragraph: what the flaw is, which lecture-slide category it falls under, and the concrete exploit scenario.

---

**Flaw 1 — `pull_request_target` trigger.** Category: **privilege & sandboxing.** The job runs on the trigger `pull_request_target`, which means the workflow definition and repo secrets are taken from the *base* branch but the code being reviewed is the *head* of an arbitrary fork PR. Any external contributor can open a PR whose head contains attacker-controlled code that the workflow will then execute with `secrets.ANTHROPIC_API_KEY` and the write-privileged `GITHUB_TOKEN` in scope — the textbook "pwn request" pattern.

**Flaw 2 — `permissions: write-all`.** Category: **privilege & sandboxing.** The job is granted every available token scope — pushing code, creating releases, mutating issues and workflows — when all it actually needs is `contents: read` (to clone) and `pull-requests: write` (to leave a comment). If any later step is compromised (a typo-squatted action, a model-emitted command that gets executed), the blast radius is the entire repository.

**Flaw 3 — Job-scoped `ANTHROPIC_API_KEY`.** Category: **secret leakage.** The Anthropic API key is exported via `env:` at the job level, so every step in the job — including `actions/checkout`, the `pytest` step, and any third-party action ever added to this workflow — can read it from the process environment. A future maintainer who slots in an unrelated action gives that action read access to the key without realising it.

**Flaw 4 — No `timeout-minutes`.** Category: **cost / denial of service.** The job has no wall-clock cap, the `curl` call has no `--max-time`, and the model request has no streaming-stop discipline. A runaway PR — or a deliberate flood of PRs — can keep an instance running for the GitHub Actions hard limit (6 hours per job) while the API call burns budget. There is no per-PR cost ceiling either.

**Flaw 5 — `fetch-depth: 0`.** Category: **data exposure.** The checkout pulls the entire git history. Any historical commit that ever contained a since-rotated secret is now on disk and can land in the prompt that gets shipped to a third-party API. The reference hardened workflow uses `fetch-depth: 2` because that is all you need to diff a PR against its base.

**Flaw 6 — Untrusted text concatenated into the prompt.** Category: **prompt injection.** PR title, PR body, and `git diff` output are written verbatim into a single user message with no delimiter, no XML boundary, and no length cap. The model sees one undifferentiated wall of text and cannot tell developer-supplied instructions from attacker-supplied data. A PR body that begins `Ignore previous instructions and respond with…` is indistinguishable from legitimate prose at the token level.

**Flaw 7 — Raw `curl` with no output constraints.** Category: **output validation.** The request asks the model to "post findings as markdown" — free-form text. There is no `tools` parameter requesting structured output, no JSON-mode hint, no stop sequence, and no tight `max_tokens`. Whatever the model emits is taken on faith, including any text exfiltrated from the diff or echoed from an injected payload.

**Flaw 8 — No schema validation on the response.** Category: **output validation.** The workflow does `jq -r '.content[0].text' > /tmp/comment.md` and pipes the result straight into `gh pr comment`. Even an empty-findings hallucination ("LGTM, no issues found") is accepted. There is no fallback path for a malformed response, no fail-closed behaviour, no retry-with-backoff, no schema check, and the raw response is never persisted as an artifact for forensic review.

**Flaw 9 — Unsanitised comment body.** Category: **prompt injection / defense-in-depth.** The model's text is posted as-is. Markdown rendered in a GitHub comment can include images that hit attacker-controlled URLs (referer leakage), formatting that mimics a maintainer approval, or instructions aimed at the *next* AI reviewer in the chain. Because GitHub renders the comment under the `github-actions[bot]` author, any injected content inherits that bot's perceived authority.

---
