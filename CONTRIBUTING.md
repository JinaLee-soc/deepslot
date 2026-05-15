# Contributing

This project is in early development. Contributions are welcome, especially around:

- Evaluation metrics and matching heuristics
- Calendar export parsers for different platforms
- Privacy-preserving feature extraction
- Policy feedback aggregation strategies

## Guidelines

- Keep the system local-first. Do not introduce cloud dependencies without an explicit opt-in flag.
- Calendar writes must be add-only and idempotent. Never delete or overwrite existing user events.
- Raw calendar data (titles, descriptions, attendees, locations) must not appear in public outputs, logs, or committed files.
- If using an LLM, it must only receive aggregate features and return bounded scoring advice. It must not write calendar events directly.
- Tests should use synthetic calendar data, not real exports.
- Any matching or evaluation heuristic must degrade gracefully when project labels, proposal IDs, or manuscript traces are missing.

## Structure

```
config/              Local configuration (gitignored except examples)
input/               Raw calendar exports (gitignored)
parsed/              Parsed calendar data (gitignored)
output/              Generated reports and policy files (gitignored)
logs/                Runtime logs (gitignored)
```

## Running locally

```bash
cp config/config.example.yaml config/config.yaml
# Edit config.yaml with your calendar taxonomy

python time_audit.py --input calendar_export.tsv
python propose_blocks.py --lookback-days 21 --lookahead-days 7 --dry-run
```
