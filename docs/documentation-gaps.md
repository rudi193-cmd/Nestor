# Documentation gaps — standing up from scratch

*Found 2026-08-05 by following README.md and docs/ verbatim, with no prior knowledge of the project.*

---

## How this was done

Followed the README from top to bottom in a clean container:

```
git clone … && cd Nestor
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
pytest -q
python demo/sixty_seconds.py --fast
```

Then exercised every CLI command and code snippet in the README, and tried each optional extra (`[keys]`, `[cloud]`, `[semantic]`).

**What worked without issue:** `pip install`, `pytest -q` (445 passed, 8 skipped), the inline Python examples, the demo script, the `nestor-ui` server, `nestor stats`, `nestor ledger verify`, `nestor export`, `nestor keys`, `nestor rejections`, `nestor calibrate`, `bench/serve_ui.py`, all three optional extras.

---

## Gaps found

### 1. `NESTOR_SEAL_KEY` — no generation instructions

The README says "Set `NESTOR_SEAL_KEY` and every seal is bound to a key the store does not hold" but never says:

- What format the value should be (random hex, passphrase, UUID, arbitrary string?)
- How long it should be
- How to generate one

Looking at `nestor/signing.py`, the env var is read as `env.encode()`, so any string works as an HMAC key — but an operator who doesn't know that is likely to use a weak value or spend time hunting for a "required format."

**Suggested addition** (Quick Start or Seal signatures section):

```bash
# generate once; keep it out of source control
python -c "import secrets; print(secrets.token_hex(32))"
export NESTOR_SEAL_KEY="<that value>"
```

---

### 2. `ruff` and `bandit` not installed by `pip install -e ".[dev]"`

The Development section says:

```bash
pip install -e ".[dev]"
ruff check nestor tests
bandit -r nestor -ll -q
```

But `pyproject.toml` only puts `pytest` in `[dev]`. After following the docs literally, `bandit: command not found`. `ruff` happened to be pre-installed in this environment (system-wide), but a clean venv would miss it too.

**Fix:** add `ruff` and `bandit` to the `[dev]` optional-dependencies in `pyproject.toml`, or add a separate `pip install ruff bandit` step in the Development section.

---

### 3. Project layout omits `bench/serve_ui.py` and `bench/ui/`

The project layout table lists bench files but stops at `harness.py` and `results/`. The Accuracy section then says:

```bash
python bench/serve_ui.py --open       # http://127.0.0.1:8770/ui/
```

`bench/serve_ui.py` and the `bench/ui/` subdirectory it serves exist and work, but they appear nowhere in the layout. A reader scanning the layout for what `bench/` contains will miss them.

---

### 4. `docs/` directory has two unlisted files

The project layout mentions only:

```
docs/code-review-lessons.md
docs/fleet-integration-map.md
```

The actual directory also contains `docs/decision-memory.md` and `docs/local-fleet.md`. Both are linked from `docs/fleet-integration-map.md` and contain substantive content, but a reader who only follows the README won't know they exist.

---

### 5. CLI examples assume a pre-populated database

The CLI section opens with:

```bash
nestor ask "Good evening."               # ✓ sealed  Buenas noches.  (verified by rita)
```

A reader who just ran the Quick Start gets:

```
! pending  —
```

The inline Python example earlier in the README seals "Good evening." into a `:memory:` store that immediately vanishes. There is no documented step that goes from Quick Start to a persistent database with a sealed pair in it. A new user has no way to reproduce the commented output without reading ahead and inferring they need to run the entities or reconcile Python examples against a file-backed store.

**Suggested addition:** one brief note before the CLI table, e.g.:

> The examples below assume a populated database. To seed one quickly, run `demo/sixty_seconds.py` with a file-backed store, or use `nestor ui` to seal your first pair.

---

### 6. `--db` flag must precede the subcommand — not mentioned

The help text shows `--db DB` as a global flag at the `nestor` level:

```
nestor [--db DB] {ask,resolve,...}
```

But the CLI examples show only the subcommand form (`nestor ask "…"`) with no `--db`. A new user who tries:

```bash
nestor ask "Good evening." --db mydb.db
```

gets `error: unrecognized arguments: --db mydb.db`. The correct form is:

```bash
nestor --db mydb.db ask "Good evening."
```

This is easy to get wrong and should be noted once, either in the CLI section or in the help text for each subcommand.

---

### 7. `SEAL_THRESHOLD` is a "dial" with no documented dial-turning mechanism

The Accuracy section describes `SEAL_THRESHOLD` as an intentionally exposed dial and instructs readers to calibrate it against their corpus. `nestor calibrate` reports the recommended value. But neither the README nor the CLI shows how to *apply* a new value:

- There is no env var for it.
- There is no CLI flag.
- The only way is `import nestor.memory; nestor.memory.SEAL_THRESHOLD = 0.96` in Python.
- Per-call override (`seal_threshold=` kwarg on `best_sealed`) exists but is not documented.

A reader who runs `nestor calibrate` and gets `recommended: 0.96` has no documented path from that recommendation to a changed serve behavior.

**Suggested addition:** a line in the Accuracy section and/or the `nestor calibrate` output pointing to how to apply the result.

---

### 8. `nestor ask --engine auto` needs `ANTHROPIC_API_KEY` — not stated

The `[cloud]` extra (`pip install -e ".[cloud]"`) and `--engine claude` / `--engine auto` are documented, but nowhere is it stated that `auto` or `claude` require `ANTHROPIC_API_KEY` to be set. The engine falls back to `offline` silently when the key is absent (the code checks `os.environ.get("ANTHROPIC_API_KEY")`), so a user who installs `[cloud]` and passes `--engine auto` without the key gets offline behavior with no explanation.

---

### 9. MCP server config block doesn't name the client application

The "Serving a model" section shows:

```json
{"mcpServers": {"nestor": {"command": "nestor",
                           "args": ["serve", "--db", "data/nestor.db"],
                           "env": {"NESTOR_SEAL_KEY": "…"}}}}
```

This JSON goes somewhere, but the docs don't say where. A first-time MCP user doesn't know this belongs in Claude Desktop's config file (or an equivalent). A one-line pointer — "add this to your MCP client's server configuration, e.g. `~/.config/claude/claude_desktop_config.json` on macOS" — would close the gap.

---

### 10. `bench/bench_accuracy.py --probes 400` takes a very long time, with no warning

The Accuracy section says:

```bash
python bench/bench_accuracy.py --probes 400
```

With `--probes 5` (the minimum practical check) the benchmark exceeded a 120-second timeout. With `--probes 400` a user might wait 20–30 minutes. There is no estimate of runtime, no "this is a slow operation" note, and the `bench/README.md` doesn't address it either. A `--probes 10` default-for-quick-check note would help.

---

### 11. WAL backup warning is easy to miss

The WAL backup limit is documented ("a plain `cp` of `nestor.db` is not a backup of a running server"), but it appears in the second paragraph of Quick Start — before users have even run anything — and is not repeated near the UI launch instructions where the risk is highest. A new operator who reads the docs top-to-bottom may have forgotten it by the time they're running `nestor-ui` in production.

`docs/code-review-lessons.md` has thorough treatment of this (§1) but is not linked from the relevant README section.

---

## What did NOT need any documentation help

- `pip install -e ".[dev]"` — worked exactly as written.
- `pytest -q` — all 445 tests passed, 8 skipped, zero surprises.
- `python demo/sixty_seconds.py --fast` — ran and self-asserted cleanly.
- The inline Python snippets (translation, entity resolution, numeric reconciliation) — all produced the exact output shown.
- `python -m nestor.ui` and `nestor-ui` — server started, served HTML, `--open` flag is present and documented.
- `nestor ledger verify`, `nestor stats`, `nestor export`, `nestor rejections`, `nestor keys add` — all worked as documented.
- `nestor calibrate --from en --to es` — the `--from` / `--to` aliases work (they are aliases for `--source-lang` / `--target-lang`, which the help text confirms but the README only shows the short form).
- `bench/serve_ui.py` — started, served the dashboard at the documented URL.
- All three optional extras (`[keys]`, `[cloud]`, `[semantic]`) installed and imported cleanly.
