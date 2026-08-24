# Cross-Agent Sync

Cross-Agent Sync gives Claude Code and Codex a small, inspectable handoff surface without turning either harness's private transcripts into shared project state.

It reads local JSONL logs, creates a bounded local-only evidence packet, and maintains two project files:

```text
.agent-sync/
├── .gitignore
├── events.jsonl       # append-only curated deltas
├── PROGRESS.md        # deterministic human-readable view
└── imports/           # raw excerpts; always ignored by Git
```

## Why this boundary

Raw agent logs mix useful context with personal text, local paths, tool output, and instructions that were valid only in one session. Cross-Agent Sync treats those logs as evidence to inspect, not authority to act. Only a concise human- or agent-curated delta belongs in version control.

The script does not call a model, upload data, or contact a service. It reads files on the local machine and writes only inside the selected project.

## Quick start

Requirements: Python 3.10+, Git, and macOS or Linux. `rg` is optional but speeds up query filtering.

Install the skill:

```bash
npx skills add AntreasAntoniou/cross-agent-sync
```

Then run the local transport from the installed repository or a clone:

```bash
python3 scripts/agent_sync.py doctor --project /path/to/project
python3 scripts/agent_sync.py sync --project /path/to/project --query project-name --days 14
```

Then inspect `.agent-sync/PROGRESS.md` and the new file under `.agent-sync/imports/`.

Record only a verified summary:

```bash
python3 scripts/agent_sync.py update \
  --project /path/to/project \
  --source codex \
  --summary "Verified the migration against the current repository." \
  --evidence "Tests passed at commit abc1234." \
  --next "Review the remote deployment."
```

See [SKILL.md](SKILL.md) for the complete operating contract and [session-formats.md](references/session-formats.md) for the supported log records.

## Privacy model

- Session discovery is local-only.
- Imported packets may contain transcript text and absolute paths. They are forced under `.agent-sync/imports/` and ignored by Git.
- The committed progress view uses the project directory name, not its absolute path.
- The tool excludes reasoning, tool calls/results, system material, generated instructions, and known notification wrappers.
- It cannot identify every secret in free-form human text. Review the curated delta before committing it.
- An import never grants authority for an external action.

## Determinism and failure behavior

- Event IDs are derived from semantic content, so retries are idempotent.
- Event JSON uses stable key ordering; progress sections have stable ordering and bounded history.
- The rendered timestamp is the latest event timestamp, so unchanged input renders byte-for-byte identically.
- Source JSONL is best-effort: malformed lines are skipped.
- Curated JSONL is canonical: malformed or unsupported events stop rendering rather than disappearing silently.
- Edits inside generated markers fail closed. Text outside the markers is preserved.
- Concurrent ledger appends use a POSIX advisory lock and an atomic replace for rendered output.

## Platform limits

The current implementation uses `fcntl.flock`, so Windows is not supported. Claude Code and Codex may change their internal log formats; use `doctor`, keep imports bounded, and update the parser when discovery stops matching. The tool supports the formats documented in this repository, not every historical or future harness build.

## Development

```bash
python3 -m unittest discover -s tests -v
python3 -m compileall -q scripts tests
git diff --check
```

## License

MIT. See [LICENSE](LICENSE).
