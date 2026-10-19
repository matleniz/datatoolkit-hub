# Step groups (#249): evidence from real use, 2026-10-19

Read-only scout over Matteo's agent chats (`~/.claude/projects/-home-matleniz--datatoolkit-agent-sdk-cwd/*.jsonl`),
his workspaces (`~/.datatoolkit/workspaces/`) and the `parkison` notebook export. Contract: comment on datatoolkit-issues#249.

## What a request becomes
- 15 unique proposals in two sessions: 6 are a single step (often a cleanup `drop_columns`), 9 are 3 to 11 steps (median 6).
- A "go" often gives one large proposal, then a cleanup `drop_columns` in a separate one.
- Exploration sessions lay down many temporary `eval_*` formula steps, then remove them by index.
- Recurring combinations: computed column + drop of temporaries (4x); group_agg train + group_agg test + formula, then drop;
  add column + reorder target last (3x, the example of the issue); round + cast + drop + rename + reorder ("back to original formats");
  onehot + rename + formula `gene_missing`.

## `parkison`: 63 steps = 12 intentions
| Steps | Intention |
|---|---|
| 1 | Drop `time_since_diagnosis` from test |
| 2-6 | Impute `age_at_diagnosis` (train) |
| 7-16 | Impute `ledd` (rounding and catch-up drop included) |
| 17-20 | Flags and filling of `time_since_intake` delays |
| 21-31 | Estimate missing `off` (`off_est`), clean up, cap at 132 |
| 32-34 | One-hot `gene` |
| 35-41 | Original formats, `off_est` -> `off`, flag `is_off_est`, target last |
| 42 | Encode `cohort` |
| 43-48 | Fix `age_at_diagnosis` in test |
| 49 | Drop the `Index` column |
| 50-52 | Add `disease_duration` (reorder and drop `gene_missing` included) |
| 53-63 | Patient-level features (OFF/ON means, age at first visit, visit count) |

The agent's own notes already quote step ranges ("steps s8864f350 to s356b1ddd" for the original formats).
The notebook export has 126 step cells out of 129; one cell per group would give 12.

## Wording
Matteo says "impute X", "one hot on gene", "encode cohort in 0, 1", "put the formats back", never `group_agg` or `formula`.
Group name = short verb phrase ("Impute ledd"); intent = the agent note's sentence, which already holds the why.
