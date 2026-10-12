<!-- From datatoolkit-issues#183 (agent-review-lead), committed by the coordinator. -->
Report for order #183 (Matteo, 2026-10-13): what the in-Studio agent does in real use, and what to change. Written by the agent-improvement lead (engine worktree `agent-review-lead`).

## Sources and method

Read-only. A Python script parsed the four `agent-sdk` transcripts in `~/.claude/projects/-home-matleniz--datatoolkit-agent-sdk-cwd/`. It extracted user turns, tool calls and their errors, per-call usage, `compact_boundary` events, the size of each tool result, and the size of each `propose_steps`. The workspace files in `~/.datatoolkit/workspaces/` were parsed for step ops, notes and memory. `~/.datatoolkit/agent` holds only `runtime.json`; there is no chat log. The engine code was checked at `599bcdc` and Studio at `81230ba`. This report gives patterns and counts only, no data values.

| Transcript | Date | User turns | Assistant calls | Max context | Compactions | Steps proposed |
|---|---|---|---|---|---|---|
| 21e2cb10 | 10-03 | 2 | 21 | 25k | 0 | 0 |
| 26cc107e | 10-03 | 4 | 37 | 45k | 0 | 2 proposals |
| 5b24693b | 10-05 | 13 | 152 | 136k | 0 (before #151) | 20 proposals |
| 621e3015 | 10-12 | 29 | 327 | 67k | **4** | 17 proposals; the workspace went from v1 to **v48** |

The 10-12 session is the main evidence. In it, Matteo prepares a longitudinal dataset for a challenge, column by column. He makes about 20 decisions about missing values and encodings.

## (a) What does not work: bugs

1. **Formula silently turns a text column into NaN.** `formula.py:612` coerces every column with `pd.to_numeric(errors="coerce")`. So `isna(<text column>)` returns 1 on every row, without any error. The agent found this by trial and wrote it into its compaction summaries as a "quirk".
2. **`propose_steps` fails with `no_studio` while Studio is open.** Two proposals of 9 steps each failed at once (`ui_bridge.send_command` when `target_session` has no listener at that instant). The agent told Matteo to reload Studio and say "go" again, and the work was redone a turn later. `send_command` fails as soon as the SSE listener is missing, with no grace for an EventSource that is reconnecting.
3. **`source_columns` error is unreadable.** When `kind` is missing, the agent gets the raw pydantic message `Unable to extract tag using discriminator 'kind'`. (Minor.)
4. **The agent cannot see what the ops can do.** `list_transforms` without `detail` keeps only `op`, `title` and `needs_target` (`tools.py` `compact_catalog`). The agent therefore saw `impute / Impute`, and never learnt that `impute` has `group_mean` / `group_prev` / `group_interp` (per entity, fitted on the frame being filled), `formula`, `fallback` and `add_indicator`. On 10-12 it never opened `transform_schema impute`. It rebuilt the same logic by hand (see feedback 1). Its own summary states "there is no interpolate op", which is wrong.

## (b) Missing features, from what Matteo asked for

- **Text comparison in formulas.** A test on a category value is refused ("text constant not allowed"). This produced 2 errors, and then a separate encoding step was needed first.
- **One-hot with "missing" as a category.** Asked on 10-05 and again on 10-12. `onehot` makes no column for NaN.
- **Keep the label column last.** Asked twice ("put the target back as the last column"). Every new column lands after it, so a `reorder_columns` step is needed each time.
- **Keep the original dtypes.** Asked on 10-12 ("respect the original formats"). Imputed integer columns become float, so `round` and `cast` steps had to be added afterwards.
- **Interpolation within an entity.** Asked on 10-05 and 10-12. It already exists (`impute group_interp`), but it was not discoverable (bug 4).
- **Long reference text that survives compaction.** Matteo pasted an 11.7k-character challenge brief twice in the 10-12 session (see feedback 2). Pasted text cannot become a workspace document: `keep_attachment` only takes file attachments.

## Feedback 1: "too many steps; the draft should be used for that"

**Measured.** The agent *does* explore in memory. On 10-12 it made 64 `evaluate` calls and 7 `preview_steps` calls (138 trial steps in total), and it materialised 48 steps.

The 48 steps break down into 26 `formula`, 10 `group_agg`, 6 `drop_columns`, 2 `onehot`, 2 `rename`, 1 `cast` and 1 `reorder_columns`. Proposals hold 1 to 9 steps, and one user decision becomes 4 to 9 steps.

**Why.** The draft tools are used. What makes the materialised list long is the recipe the agent builds:

1. **Recipes rebuilt by hand** (bug 4). "Fill a column from the same patient's other rows, with a flag" is a single `impute` step (`strategy: group_mean | group_interp`, `by`, `order`, `fallback`, `add_indicator`). The agent instead used `group_agg` + `formula` (coalesce) + `formula` (flag) + `drop_columns` (helper column).
2. **Per-split duplicates.** `group_agg` is fitted on train, so test entities get NaN. The agent learnt to write one `group_agg` with target train and one with target test: 10 `group_agg` steps, which are 5 pairs. The `impute` group strategies do not have this problem: they fill from the frame being transformed.
3. **Helper columns.** 3 derived columns were created and then dropped by later steps. The agent said so itself: "it remains to drop the intermediate columns".
4. **Follow-up fix steps** for dtype, column order and encoding (see (b)).
5. **Text workarounds** (bug 1 and the formula text constant).

**Recommendation (no new draft store for now).**

- Fix discovery: the compact `list_transforms` keeps a one-line description.
- Rewrite the step rules in the system prompt:
  - one step per user decision, using the highest-level op (`impute` group strategies, `add_indicator`);
  - no helper columns in the pipeline: compute them in `preview_steps` / `evaluate`, then propose the final single step;
  - when a run of the agent's own recent steps can be collapsed, propose `replace` / `remove` by id.
- Close the gaps that force extra steps: onehot missing category, text equality in formulas, formula refusing text columns.

A server-side draft is the larger option: proposals pile up in a pending draft, and the user applies them as a batch or squashed. It changes the Studio UX Matteo validated, so it is a decision for him (below). I recommend measuring first: on 10-12, the five items above account for about 30 of the 48 steps.

## Feedback 2: context and memory

**How each pack manages context today.**

- `agent-sdk` passes `CLAUDE_CODE_AUTO_COMPACT_WINDOW = DTK_AGENT_COMPACT_AT` (default 60k, #151). The CLI then replaces the whole thread with a summary. There is nothing else: no clearing of stale tool results, no workspace re-intro after a compaction.
- `api-anthropic` / `api-openai` (`api_chat.py`) keep the last 4 user turns whole and elide older tool results. The oldest turns are dropped above 400k chars. No transcripts exist for these packs.
- All packs send a per-turn Studio note (view and step diff), plus a one-time intro per conversation (analysis keys, documents, memory). Since `chat.introduced` is not reset on compaction, the intro is lost with the first summary.

**Measured on 10-12.** There were 4 auto-compactions, at 67-69k context each (after 20, 15, 155 and 6 minutes). Each one cut the context to about 3k tokens and took 28 to 40 s, during which Matteo waited. The cumulative dropped total was 261k tokens.

What was lost:

- **The pasted challenge brief.** Three minutes after compaction 1, Matteo asked "did you keep the challenge rules in memory?" The agent answered that it no longer had his original text and had reconstructed it. Matteo re-pasted the 11.7k chars.
- **Column names it had created.** The first tool call after compaction 1 failed on an unknown column name.
- **Engine knowledge.** After compactions, `transform_schema` was fetched again (13 calls in the session), and each of the four summaries spends a large share re-listing engine quirks.
- **The thread.** After a long pause (and the `no_studio` failure), Matteo pasted the agent's own earlier question back with "resume from there".

What filled the context: `get_workspace` took about 16k chars per call (4 calls, 66k), `get_notes` about 15k per call (2 calls, 30k) and `evaluate` 79k over 64 calls. Large reads were the main growth.

**Recommendation.**

1. *Engine, no product change:* after a compaction (`compact_boundary`), the next turn re-sends the workspace-state intro: keys, documents, memory and the full step list as for a first turn, plus the head of the workspace note. The agent then restarts from the workspace state and not just from a thread summary. The api packs get the same treatment when they drop turns.
2. *Engine:* make heavy reads lighter. `get_workspace` returns step lines and note presence by default, with full params on demand. `get_notes` returns capped texts with a `full` flag.
3. *Prompt:* when the user pastes long reference material, or asks to keep it, the agent offers to keep it as a workspace document. It keeps its working facts (created columns, engine quirks it learnt) in memory, not in the thread.
4. *Decision for Matteo:* make the threshold dynamic, e.g. a default tied to the model window (such as 50% of it) or a higher fixed default. Keep `DTK_AGENT_COMPACT_AT` as an override.
5. *Decision for Matteo:* let Studio turn a long paste (over N chars) into an attachment automatically, so it can be kept as a document.

## Feedback 3: notes and comments are not written by themselves

**Measured.** On 10-12 there were 39 `set_note` calls, but they came on request. After "mark this in a note" (09:52) and "add notes to justify the previous points" (12:43), there was a burst of **17** `set_note` calls written after the fact. Only 24 of the 48 steps carry a note. Before 12:43, proposals of 8 and 9 steps came with at most one note.

**Why.** `propose_steps` cannot carry a note. The step schema (`commands.py _STEP`) has only `op` / `target` / `params`, and Studio's `parseStep` drops anything else. So a note takes one extra `set_note` round trip per step after the ack. For a destructive proposal (`pending: "review"`), the agent has no step id, so it cannot note it at all, and the prompt tells it to end its turn.

**Recommendation: bridge behaviour plus prompt, not prompt alone.**

- An optional `note` on each proposed step: engine schema and docs; Studio keeps it on the step and shows it in the review.
- The prompt makes it the rule: every added step has a one-line "why" (with the measured number when a study chose it). The agent sets a column note when it creates or redefines a column. The workspace note gets a short log line when a decision closes a topic.

The note is part of the proposal the user sees and reviews, so nothing is written behind the user's back. Auto-generating notes engine-side from a separate "rationale" is not needed.

## Deliverables (sub-issues of the epic)

| # | Repo | What | Decision |
|---|---|---|---|
| D1 | engine | compact `list_transforms` keeps a one-line description | no |
| D2 | engine | system prompt: step discipline, notes on every step, keep pasted references, memory for working facts | no |
| D3 | engine + web | optional `note` on proposed steps (schema, Studio keeps it and shows it in review) | no |
| D4 | engine | re-send the workspace-state intro after a compaction (agent-sdk) or a history drop (api packs) | no |
| D5 | engine | formula refuses text columns clearly; equality / inequality against a text constant | no |
| D6 | engine | `send_command` waits a short grace for a reconnecting Studio listener before `no_studio` | no |
| D7 | engine | `onehot` option: missing as its own category | no |
| D8 | engine | lighter `get_workspace` / `get_notes` payloads | no |
| D9 | engine | dynamic compaction threshold | **needs:matteo** |
| D10 | web | long paste becomes an attachment (to keep as a document) | **needs:matteo** |
| D11 | engine + web | a real pending draft (batch / squash before materialising) | **needs:matteo**; recommend: measure after D1-D7 first |
| D12 | engine | keep the label column last and preserve integer dtypes on fill | low priority, after D1-D7 |

