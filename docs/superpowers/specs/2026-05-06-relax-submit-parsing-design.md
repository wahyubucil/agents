# Relax Submit-Prompt Parsing in `/review` and `/review-discussion` — Design

## Goal

Stop rejecting natural-language replies at the "Submit?" prompt in `/review` and `/review-discussion`. Today the parser accepts only the literal tokens `n` / `comment` / `request_changes` / `approve` (plus a couple of variants). A reply like "cool please submit" — sent after the user has already read the pending review — gets bounced with `Got "cool please submit", which is not n/comment/request_changes/approve. Try again.` even though the intent is unambiguous.

The change makes the parser accept loose phrasing for the three submission events while preserving the safety guarantee that the model never auto-submits without a user reply at the prompt.

## Motivation

The current strict parser was put in place to keep the model from sycophantically inferring a submission event from review tone or from a pre-emptive "submit it" in the original question. That guard is still useful for the *question phase* — but it overshoots once the pending review is on GitHub and the user has actually evaluated it.

By the time the "Submit?" prompt appears, the user:

1. Has seen the pending review on GitHub (the orchestrator prints the URL).
2. Has read the inline comments and the draft body.
3. Is now choosing what to do with the review they just inspected.

In that state, "submit", "send it", "go ahead" are all unambiguous: the user wants to submit, and given the inline comments are flagging issues, REQUEST_CHANGES is the only honest event. The current parser pretends not to know that and bounces the reply, forcing the user to re-type a strict keyword they often don't remember (`request_changes` underscore vs space, etc.).

We keep the model's mouth shut at the question phase (no auto-submit even on "submit it"), but loosen the parser at the reply phase.

## Scope

Markdown-only edits to two files:

- `plugins/code-review/commands/review.md`
- `plugins/code-review/commands/review-discussion.md`

No code changes, no new files, no test framework. These are LLM agent prompts; verification is `grep` for stale fragments and re-reads for internal consistency.

## Out of scope

- Changing the posting CLI (`gh-pr-review`) or its flags.
- Changing the review-composition flow upstream of step 14 (`/review`) / step 5 (`/review-discussion`).
- Letting the model auto-submit at the question phase. The "no pre-emptive auto-submit" rule stays.
- Inferring the event from review content (the rejected Approach C from brainstorming).

## Design

### 1. Parser logic

Replace the strict-keyword parser in:

- `review.md` step 14, the "Parse the user's reply..." block (currently lines 503–514).
- `review-discussion.md` step 6, the "First non-empty line..." block (currently lines 470–480).

New parser, applied case-insensitively to the **first non-empty line**, trimmed; **first match wins**:

| Reply matches | Resolved event |
|---|---|
| empty, `n`, `no`, `nope`, `cancel` | exit, keep pending review on GitHub |
| line contains `approve`, `lgtm`, `ship it`, `looks good`, `good to go` | `APPROVE` |
| line is exactly `comment`, `comments`, or `comment only` | `COMMENT` |
| line contains `submit`, `send`, `post`, `request changes`, `request_changes`, `go ahead` | `REQUEST_CHANGES` |
| anything else | re-ask |

Order matters: the approve check runs **before** the request_changes check, so "approve and submit" → APPROVE. The cancel match is exact (line is one of `n` / `no` / `nope` / `cancel` after trim) — a sloppy "no, send it" falls through to rule 4 and submits as REQUEST_CHANGES. If the user means cancel, they should type just `n` or `no`.

The "comment" row is intentionally an *exact-match* set, not a `contains` check — too easy for "submit a comment" or "approve, just one comment on line 4" to false-match otherwise.

Lines 2+ remain the body verbatim. No change to body parsing.

### 2. Submit? prompt copy

Replace the current option list in both files. The two prompts share a parser but diverge on body handling: `/review-discussion` accepts the body verbatim from lines 2+ of the reply, while `/review` uses the pre-composed draft body from step 13 (the user reply is event-only).

`review.md` step 14 (currently lines 496–499):

```
Submit? Reply with one of:
  n / no / cancel                          — don't submit (review stays pending on GitHub)
  approve / lgtm / ship it / looks good    — submit as APPROVE with the draft summary above as the body
  comment                                  — submit as COMMENT with the draft summary above as the body
  submit / send / post / request changes   — submit as REQUEST_CHANGES with the draft summary above as the body
```

`review-discussion.md` step 5 (currently lines 460–463):

```
Submit? Reply with one of:
  n / no / cancel                          — don't submit (review stays pending on GitHub)
  approve / lgtm / ship it / looks good    — submit as APPROVE
  comment                                  — submit as COMMENT (non-blocking)
  submit / send / post / request changes   — submit as REQUEST_CHANGES
Lines after the first become the summary body verbatim. Omit for an empty body.
```

Body-handling parity is intentionally **not** introduced here — that would silently change `/review`'s behavior (auto-composed structured summary → user-replaceable). Out of scope for this change.

### 3. Re-ask copy

Replace the current re-ask block in both files (review.md:511-514, review-discussion.md:477-480):

```
Got "<first line>", which I can't read as cancel/approve/comment/submit. Try again with one of:
  n          — don't submit
  submit     — submit as REQUEST_CHANGES (also "request changes", "send", "post")
  approve    — submit as APPROVE (also "lgtm", "ship it", "looks good")
  comment    — submit as COMMENT
```

`<first line>` is substituted with the user's actual first non-empty line (truncated to ~60 chars if longer).

### 4. Hard-rule wording

The current "Hard rules" section in both files claims the parser only accepts explicit keywords. After this change that's no longer true. Reword to preserve the real invariant — *the model never submits without a user reply at the Submit? prompt* — while no longer overspecifying what counts as a valid reply.

`review-discussion.md:518` currently:

> **Never auto-submit a pending review** — even when the user pre-emptively asks in the question (e.g. "draft a pending review for what you find" or "submit it"). Always stop at the "Submit?" prompt and wait for an explicit event keyword (`comment`, `request_changes`, or `approve`) from the user. **Stop. Do not submit.**

New:

> **Never auto-submit a pending review.** Even when the user pre-emptively asks in the question (e.g. "draft a pending review for what you find" or "submit it"), the model must always stop at the "Submit?" prompt and wait for the user's reply. Step 6 defines what counts as a valid reply — loose phrasing is fine, but the reply must come from the user *post-prompt*, not be inferred from the original question. **Stop. Do not submit.**

`review.md:534` currently:

> **Never auto-submit.** Always stop at "Submit?" (step 14). The pending review must be visible on GitHub before the user has a chance to approve submission. The only path that runs `gh pr-review review --submit` is when the user replies in step 14 with an explicit event keyword (`comment`, `request_changes`, or `approve`). **Stop. Do not submit.**

New:

> **Never auto-submit.** Always stop at "Submit?" (step 14). The pending review must be visible on GitHub before the user has a chance to approve submission. The only path that runs `gh pr-review review --submit` is when the user replies in step 14 — see step 14 for what counts as a valid reply. Loose phrasing is allowed; the reply must come from the user *post-prompt*. **Stop. Do not submit.**

## Verification

After editing both files:

1. `grep -n "request_changes\|REQUEST_CHANGES" plugins/code-review/commands/review*.md` — confirm no stale references to "explicit event keyword" or to the old four-token list outside the new parser tables.
2. Re-read each file's step 14 / step 6 end-to-end and the "Hard rules" section to confirm internal consistency.
3. Manual smoke test on a real PR (optional, not blocking): run `/review-discussion` with a question that produces inline comments; reply "cool please submit" at the prompt; confirm REQUEST_CHANGES is submitted and the body is empty (since lines 2+ were absent).

No automated tests exist for these prompts; the merge gate is human review of the markdown diffs.

## Risks and trade-offs

- **False-positive REQUEST_CHANGES on confused replies.** A user replying "wait, what's in the diff?" no longer matches a known verb, so the re-ask path catches it (Approach 2 from brainstorming). Picking the looser fall-through (Approach 1) was rejected for this reason.
- **Approve-before-submit ordering.** "approve and submit" submits as APPROVE because approve is checked first. This matches user intent (the strong word wins), but worth keeping in mind if anyone files a bug saying "I typed 'approve and submit' and it approved".
- **Keyword list churn.** The submission-verb list (`submit / send / post / request changes / go ahead`) and approve list (`approve / lgtm / ship it / looks good / good to go`) are likely to grow over time as users discover phrasings that bounce. That's fine — the parser tables are easy to extend.
- **Hard-rule reword still permits one footgun.** "Pre-emptively asks 'submit it' in the question" → still must stop at the prompt. If a future contributor reads only the parser table and not the hard-rule section, they could be tempted to skip the prompt when the question contains "submit". The hard-rule wording flags that explicitly.
