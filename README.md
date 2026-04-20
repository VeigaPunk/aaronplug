# aaron

`aaron` is a minimal POSIX CLI that fetches **public-domain books** (via `lib*` mirrors) and **academic papers** (via `arxiv`, `semantic-scholar`, and `sci*` fallback). It emits compact JSON to stdout. Any agent with a shell / Bash tool — Claude Code, Codex CLI, Gemini CLI — can call it directly. No MCP server, no daemon, no wrapper.

> "We are trendsetters — we adapt tools to our needs." No wire-protocol ceremony. stdout is the contract.

---

## Install

```bash
git clone https://github.com/VeigaPunk/aaronplug
cd aaronplug
bun install
bun link          # makes `aaron` globally available on $PATH
```

Build a standalone bundle:

```bash
bun run build     # emits ./build/aaron.js (node target, shebanged)
```

Compile to a static binary (no Bun/Node needed at runtime):

```bash
bun run compile   # emits ./standalone-executables/aaron-<platform>-<arch>
```

---

## Usage

```
aaron <command> [options]

books search <query> [--format <ext>]       Search lib* mirrors
books get    <md5>   [--output-dir <path>]  Download one book
books batch  <md5-list-file> [-o <path>]    Download many books
books url    <md5>                          Resolve direct download URL

papers fetch <doi>   [--mode <tier>]        Fetch paper text
                                            tier: auto | arxiv | s2 | scihub
help                                        Show this message
```

### Output contract

- **stdout** is **compact JSON** — one object per invocation, no pretty-printing, no wrapper.
- **stderr** is human-readable diagnostics (progress, mirror failures). Most agents ignore it.
- **exit 0** = success. **exit 1** = error; stdout will be `{"error":"..."}`.

---

## Books — examples

```bash
aaron books search "tolstoy war and peace"
# → {"query":"tolstoy war and peace","count":7,"results":[{...},{...}]}

aaron books get 13c11d86028143eccc200e2e31af8511 -o ./downloads
# → {"md5":"...","path":"./downloads/...","filename":"...","size":1234567}

aaron books batch ./MD5_LIST.txt -o ./downloads
# → one {"md5":...,"path":...} line per entry

aaron books url 13c11d86028143eccc200e2e31af8511
# → {"md5":"...","url":"https://..."}
```

Mirror list is fetched from the upstream configuration: `https://github.com/VeigaPunk/aaronplug/blob/configuration/config.v3.json`.

---

## Papers — three-tier cascade

`aaron papers fetch <DOI>` tries sources in order and returns the first that succeeds:

| order | tier     | source              | format    | best for                         |
| ----- | -------- | ------------------- | --------- | -------------------------------- |
| 1     | `scihub` | sci-hub HTML→PDF    | `markdown`| full text via `@opendocsg/pdf2md`|
| 2     | `arxiv`  | arxiv e-print tar   | `latex`   | STEM preprints (full source)     |
| 3     | `s2`     | Semantic Scholar    | `abstract`| metadata-only last resort        |

Force a single tier with `--mode arxiv|s2|scihub`.

```bash
# Auto cascade — tries arxiv, then s2, then scihub
aaron papers fetch 10.1038/nature12373

# Force arxiv
aaron papers fetch 10.48550/arXiv.1706.03762 --mode arxiv

# Force full-text via sci-hub + PDF→markdown (no Python; pure TS)
aaron papers fetch 10.1103/PhysRevLett.116.061102 --mode scihub
```

Response shape (always):

```json
{
  "doi": "10.1038/nature12373",
  "tier": "arxiv",
  "format": "latex",
  "text": "\\documentclass{nature}\n\\title{...}\n...",
  "meta": { "arxivId": "1304.1068", "tarballBytes": 123456 }
}
```

---

## Agent integration

All three CLIs pass `stdout` verbatim to the model as a string. None of them sniff JSON — the model parses it. Therefore:

### Claude Code

```bash
# Just put aaron on $PATH; no registration needed.
bun link
# Then in any Claude Code session:
# Model invokes: Bash(command="aaron books search 'tolstoy'")
```

Note: Claude Code persists outputs >100KB to disk and hands the model a preview + path. Large sci-hub markdown outputs may trigger this — not a bug, just how the harness works.

### Codex CLI

```bash
# Same — PATH-level registration. Codex uses bash -c for tool calls.
aaron --help   # confirm on PATH
```

### Gemini CLI

```bash
# Same — invoked via Gemini's run_shell_command built-in tool.
# Optionally set a tokenBudget in settings.json if sci-hub outputs are large:
#   "toolSettings": { "run_shell_command": { "tokenBudget": 40000 } }
```

---

## Architecture

```
src/
├── index.ts                 top-level argv router (books | papers | help)
├── cli/
│   ├── books.ts             books subcommand + programmatic booksApi
│   ├── papers.ts            papers subcommand
│   └── help.ts              help text
├── api/
│   ├── adapters/            lib* mirror adapters (preserved from v3.x)
│   ├── data/                mirror config, document fetch, file download
│   ├── models/              Entry, DownloadResult types
│   └── papers/
│       ├── index.ts         cascade orchestrator
│       ├── arxiv.ts         tier 1 — DOI→arxivId→e-print tarball→tex
│       ├── semantic-scholar.ts   tier 2 — /graph/v1/paper/DOI
│       └── scihub.ts        tier 3 — HTML parse → PDF → pdf2md
├── settings.ts              constants
└── utilities.ts             `attempt` retry helper, text clean
```

---

## Origin

Forked from [`epubdomain-downloader`](https://github.com/VeigaPunk/aaronplug) (v3.3.1) by Omercan Balandi. The React/Ink TUI layer was removed; the `libgen-plus` adapter preserved. Papers support and POSIX-JSON output contract are new to aaron.

## License

WTFPL.
