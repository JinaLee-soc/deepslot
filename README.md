# Deepblock

A local-first scheduling feedback loop for academic research work.

Deepblock proposes research blocks from free/busy calendar data, evaluates whether those blocks were adopted and followed by manuscript-relevant work traces, and uses the results to update the next scheduling policy. It does not try to fill empty calendar slots. It learns which kinds of free time reliably become research progress.

## Why this exists

Many calendar tools can find open time and place focus blocks. Fewer systems ask whether those blocks actually became usable research time.

This project treats scheduling as a feedback problem:

```
time audit → proposed work blocks → actual calendar behavior →
manuscript-conversion evaluation → next-week scheduling policy
```

It is designed for researchers whose work depends on sustained writing, analysis, revision, and reading time. The main problem it addresses is not total work hours, but the conversion of available time into manuscript progress. The system does not try to fill empty calendar slots. It learns which kinds of free time reliably become research output.

## Core idea

The system separates four tasks:

1. **Analyze** past calendar structure.
2. **Propose** future research blocks from free/busy data.
3. **Evaluate** whether proposed blocks were adopted and executed.
4. **Update** the next scheduling policy using observed conversion patterns.

The key object is a proposed work block with a stable identifier:

```json
{
  "proposal_id": "rtpl_2026w21_003",
  "proposal_window": "2026-05-18_to_2026-05-24",
  "project": "Project A",
  "block_type": "manuscript_writing",
  "intended_outcome": "revise theory section",
  "start": "2026-05-22T09:00:00-05:00",
  "end": "2026-05-22T11:00:00-05:00",
  "candidate_features": {
    "weekday": "Friday",
    "duration_minutes": 120,
    "post_meeting_gap_minutes": 180,
    "fragmentation_score": 0.18,
    "anchor_match": true
  },
  "policy_version": "rtpl_policy_2026-05-15"
}
```

This makes it possible to evaluate proposed time blocks as actual interventions rather than vague calendar suggestions.

## System components

### 1. Time audit layer

Reads a local calendar export or parsed calendar data and computes descriptive features of the research week.

Possible features include: total research-labeled time, deep-work-eligible time, meeting fragmentation, meeting-to-writing lag, weekday and weekend work patterns, protected blocks, short-session clustering, project-specific traces, and recovery or overload sequences.

This layer is descriptive. It does not infer productivity from calendar labels alone.

### 2. Proposal layer

Reads calendar availability and proposes research blocks for the next planning window.

Uses only the minimum calendar data required for scheduling: calendar list metadata, free/busy intervals, optional local project configuration, aggregate time-audit features, and prior feedback summaries.

The proposal layer avoids reading source event bodies, attendees, locations, conference links, descriptions, or private event text.

Each proposed block includes: a stable `proposal_id`, project, block type, intended outcome, start and end time, candidate features, policy version, and creation timestamp.

Proposed blocks are written to a separate proposal calendar or exported as a reviewable file. They do not overwrite existing calendar events.

### 3. Evaluation layer

Compares proposed blocks with actual calendar traces after the week ends.

For each proposed block, it computes:

```json
{
  "proposal_id": "rtpl_2026w21_003",
  "adopted": true,
  "executed": true,
  "execution_overlap_ratio": 0.83,
  "protected_from_meetings": true,
  "converted_to_manuscript_trace": true,
  "conversion_trace_type": "draft_revision",
  "conversion_confidence": "medium"
}
```

| Metric | Definition |
|--------|-----------|
| `adopted` | The proposed block remained on the calendar or was manually accepted. |
| `executed` | A research, writing, analysis, or reading trace overlapped with the proposed block. |
| `execution_overlap_ratio` | Fraction of the proposed block covered by actual research-relevant calendar time. |
| `protected_from_meetings` | No meeting was inserted into or immediately adjacent to the proposed block. |
| `converted_to_manuscript_trace` | The block was followed by a project-specific manuscript trace. |
| `conversion_confidence` | Confidence level based on project matching, timing, and trace quality. |

The evaluation layer supports fallback matching when `proposal_id` is unavailable, using time overlap, project label, and block type.

**What counts as a manuscript trace.** A manuscript trace is any evidence that project-specific research work occurred. It can be supplied by the user as a local file-change log, a manually maintained project log, a writing-session export, or a structured CSV. The system does not assume that calendar labels alone prove manuscript progress. When no trace source is configured, conversion metrics are omitted rather than inferred.

### 4. Policy feedback layer

Aggregates weekly evaluation results into a small policy file:

```json
{
  "week": "2026-W21",
  "summary": {
    "proposed_blocks": 6,
    "proposed_hours": 10.5,
    "adopted_blocks": 4,
    "executed_blocks": 3,
    "converted_blocks": 2
  },
  "policy_recommendations": {
    "increase_weight": [
      "Friday morning 90-150 minute writing blocks",
      "Sunday late-morning continuation blocks",
      "blocks with at least 120 minutes after meetings"
    ],
    "decrease_weight": [
      "Monday afternoon writing blocks",
      "blocks within 90 minutes after meetings",
      "fragmented blocks under 60 minutes"
    ],
    "weekly_cap_adjustment": {
      "current_cap_hours": 12,
      "recommended_cap_hours": 10,
      "reason": "Low adoption rate for overflow blocks"
    }
  }
}
```

The next proposal run reads this policy file and adjusts candidate scoring.

## Candidate scoring

A simple scoring function is sufficient for the first version:

```
score =
  base_score
  + anchor_match_bonus
  + prior_conversion_bonus
  + project_urgency_bonus
  + sufficient_duration_bonus
  - post_meeting_penalty
  - fragmentation_penalty
  - recent_rejection_penalty
  - overbooking_penalty
```

The scoring function remains deterministic after optional policy generation. If an LLM is used, it only produces bounded scoring advice and does not directly write calendar events.

## Configuration

Copy `config/config.example.yaml` to `config/config.yaml` and define:

- allowed work days and hours
- proposal calendar name
- ignored calendars
- weekly proposal cap
- block duration bounds
- scoring weights
- active project list with per-project urgency
- optional LLM policy settings

Only projects with an active scheduling status are eligible for proposal assignment. Projects that have been submitted, are under review, accepted, or published are excluded from scheduling by default. See `config.example.yaml` for the full status taxonomy.

## Privacy model

The system is local-first and privacy-preserving.

- Do not send raw calendar titles, descriptions, attendees, locations, or conference links to cloud models.
- Use free/busy data whenever possible.
- Keep source calendar event details local.
- Store raw calendar data in ignored/private directories.
- Public reports use aggregate features only.
- Proposal events are private, transparent, attendee-free, reminder-free, and written only to a dedicated proposal calendar.
- Calendar writes are add-only and idempotent.
- All directories under `input/`, `parsed/`, `output/`, and `logs/` are gitignored by default. Outputs may contain private derived data and should never be committed.

## Data files

Recommended output structure:

```
output/
  proposed_blocks_YYYY-MM-DD.json
  proposal_log.jsonl
  proposal_write_log.jsonl
  proposal_evaluation_YYYY-MM-DD.csv
  proposal_evaluation_YYYY-MM-DD.json
  proposal_evaluation_YYYY-MM-DD.md
  policy_feedback_latest.json
  policy_feedback_YYYY-MM-DD.json
```

## CLI

```bash
# Analyze past time use
python time_audit.py --input calendar_export.tsv

# Propose next week's work blocks (dry run)
python propose_blocks.py --lookback-days 21 --lookahead-days 7 --dry-run

# Write proposed blocks to a proposal calendar
python propose_blocks.py --lookback-days 21 --lookahead-days 7 --apply

# Evaluate last week's proposed blocks
python evaluate_proposals.py \
  --proposal-log output/proposal_log.jsonl \
  --calendar parsed/calendar.parquet

# Generate next policy file
python evaluate_proposals.py --write-policy-feedback
```

## Suggested weekly workflow

```
Sunday evening:
  propose next week's research blocks

During the week:
  researcher keeps, edits, or deletes proposed blocks

Following Sunday:
  evaluate proposed blocks against actual calendar traces
  write policy feedback
  use feedback in the next proposal run
```

## Weekly report

The weekly human-readable report is short:

```
Research Time Policy Loop — Weekly Evaluation

Proposed: 6 blocks / 10.5 hours
Kept: 4 blocks / 7.5 hours
Executed: 3 blocks / 5.5 hours
Converted: 2 blocks / 3.5 hours

Best-performing pattern:
  Friday morning writing blocks with no meeting in the prior 3 hours.

Weak pattern:
  Monday post-meeting writing blocks.

Next scheduling policy:
  Reduce Monday writing proposals.
  Preserve Friday and Sunday anchor blocks.
  Cap proposed research time at 10 hours next week.
```

## Non-goals

This project does not:

- automate writing
- judge scholarly quality
- infer productivity from calendar labels alone
- read private event text unless explicitly configured
- replace human planning
- optimize for total hours worked
- schedule meetings with other people
- send calendar data to third-party services by default

## Implementation phases

### Phase 1: Proposal ledger

Generate stable proposal IDs. Write proposed block metadata to JSON and JSONL. Preserve idempotent calendar writes. Keep a policy version field on every block.

### Phase 2: Evaluation

Match proposed blocks to actual calendar traces. Compute adoption, execution, and overlap metrics. Produce weekly CSV, JSON, and Markdown evaluation reports.

### Phase 3: Manuscript-conversion linkage

Link executed blocks to project-specific manuscript traces. Add conversion confidence. Separate writing, analysis, reading, revision, and administrative research work.

### Phase 4: Policy feedback

Generate `policy_feedback_latest.json`. Feed aggregate recommendations into the next proposal run. Add scoring adjustments for high-performing and low-performing block patterns.

### Phase 5: Optional LLM policy layer

Use an LLM only to summarize aggregate evaluation data into bounded scoring advice. Do not allow the LLM to write calendar events directly. Keep deterministic validation and calendar writing outside the LLM layer.

## Policy feedback schema

```json
{
  "schema_version": "0.1",
  "generated_at": "2026-05-24T18:00:00-05:00",
  "window": {
    "start": "2026-05-18",
    "end": "2026-05-24"
  },
  "metrics": {
    "proposed_blocks": 6,
    "adopted_blocks": 4,
    "executed_blocks": 3,
    "converted_blocks": 2,
    "proposed_hours": 10.5,
    "executed_hours": 5.5,
    "converted_hours": 3.5
  },
  "pattern_weights": {
    "friday_morning": 0.25,
    "sunday_late_morning": 0.2,
    "post_meeting_under_90min": -0.3,
    "duration_under_60min": -0.15
  },
  "recommendations": {
    "preferred_windows": [
      "Friday 09:00-12:00",
      "Sunday 10:00-13:00"
    ],
    "avoid_windows": [
      "Monday after meetings"
    ],
    "weekly_cap_hours": 10
  }
}
```

## Design principle

The system optimizes for **manuscript-capable time**, not empty time.

The main question is not "when is the calendar free?" but **"which kinds of free time reliably become research progress?"**

## License

MIT
