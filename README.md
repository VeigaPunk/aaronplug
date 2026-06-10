# aaron

**A CLI that fetches any book or paper, straight to your terminal.**

`aaron` pulls down public-domain **books** (from `lib*` mirrors) and academic
**papers** (from `arxiv`, `semantic-scholar`, and `sci*`) and prints a compact
line of JSON. That's it. It was built on one idea: you should be able to
*fetch and source any resource directly from the command line* — no browser, no
sign-up, no clicking around.

```bash
aaron books search "tolstoy war and peace"
aaron papers fetch 10.1038/nature12373
```

Give it a book title or a paper's DOI; it finds it, grabs it, and hands you
back the file (or its text) plus a tidy JSON receipt.

---

## For humans

### Install

```bash
git clone https://github.com/VeigaPunk/aaronplug
cd aaronplug
bun install
bun link          # puts `aaron` on your $PATH
```

Prefer a single file or a standalone binary?

```bash
bun run build     # → ./build/aaron.js  (needs node to run)
bun run compile   # → ./standalone-executables/aaron-<platform>-<arch>  (needs nothing)
```

### The four things it does

```
aaron books search <query>     find books          → list of matches (each has an md5)
aaron books get    <md5>       download one book    → saves the file
aaron books batch  <file>      download many        → one md5 per line in the file
aaron books url    <md5>       just give me the link

aaron papers fetch <doi>       get a paper's text   → markdown / latex / abstract
```

### Examples

```bash
# Search for a book — every result includes an md5 you can download with
aaron books search "tolstoy war and peace"
# → {"query":"tolstoy war and peace","count":7,"results":[{...}]}

# Download it
aaron books get 13c11d86028143eccc200e2e31af8511 -o ./downloads
# → {"md5":"...","path":"./downloads/...","filename":"...","size":1234567}

# Grab a paper by DOI
aaron papers fetch 10.1038/nature12373
# → {"doi":"...","tier":"scihub","format":"markdown","text":"...","meta":{...}}
```

### How paper-fetching works

`aaron papers fetch <doi>` tries three sources in order and returns the first
that works:

| order | tier     | source           | you get    | best for                          |
| ----- | -------- | ---------------- | ---------- | --------------------------------- |
| 1     | `scihub` | sci-hub HTML→PDF | `markdown` | full text (via `@opendocsg/pdf2md`) |
| 2     | `arxiv`  | arxiv e-print    | `latex`    | STEM preprints (full source)      |
| 3     | `s2`     | Semantic Scholar | `abstract` | metadata-only last resort         |

Pin a single source with `--mode`:

```bash
aaron papers fetch 10.48550/arXiv.1706.03762 --mode arxiv
aaron papers fetch 10.1103/PhysRevLett.116.061102 --mode scihub
```

---

## For agents

`aaron` is designed to be a **plug** for any agent that has a shell/Bash tool —
Claude Code, Codex CLI, Gemini CLI. There is **no MCP server, no daemon, no
wrapper to register.** The contract is simple: the agent runs `aaron` as a shell
command, and `aaron` writes one JSON object to **stdout**. The model parses it.

### Wiring aaron into your agent

The whole setup is putting `aaron` on `$PATH`:

```bash
bun link          # or drop a compiled binary somewhere on $PATH
aaron --help      # confirm it resolves
```

That's the entire integration. Once `aaron` is on `$PATH`:

- **Claude Code** — model invokes `Bash(command="aaron papers fetch 10.1038/nature12373")`.
- **Codex CLI** — same; Codex runs tool calls through `bash -c`.
- **Gemini CLI** — same; invoked via the built-in `run_shell_command` tool.

### The output contract

- **stdout** is **compact JSON** — exactly one object per invocation, no
  pretty-printing, no envelope. Feed it straight to the model.
- **stderr** is human-readable diagnostics (progress, mirror failures). Agents
  can ignore it.
- **exit 0** = success. **exit 1** = error, and stdout will be
  `{"error":"...message..."}`.

Uniform paper response shape:

```json
{
  "doi": "10.1038/nature12373",
  "tier": "arxiv",
  "format": "latex",
  "text": "\\documentclass{nature}\n\\title{...}\n...",
  "meta": { "arxivId": "1304.1068", "tarballBytes": 123456 }
}
```

### Notes per harness

- **Claude Code** persists outputs >100KB to disk and hands the model a preview
  plus a path. Large sci-hub markdown can trip this — expected, not a bug.
- **Gemini CLI** — if sci-hub outputs are large, raise the budget in
  `settings.json`:
  `"toolSettings": { "run_shell_command": { "tokenBudget": 40000 } }`.

---

## Reference

```
aaron <command> [options]

books search <query> [--format <ext>]       Search lib* mirrors
books get    <md5>   [--output-dir <path>]   Download one book
books batch  <md5-list-file> [-o <path>]     Download many books
books url    <md5>                           Resolve direct download URL

papers fetch <doi>   [--mode <tier>]         Fetch paper text
                                             tier: auto | arxiv | s2 | scihub  (default: auto)
help                                         Show this message
```

The lib* mirror list is fetched from upstream config:
`https://github.com/VeigaPunk/aaronplug/blob/configuration/config.v3.json`.

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
│       ├── index.ts         cascade orchestrator (scihub → arxiv → s2)
│       ├── arxiv.ts         DOI → arxivId → e-print tarball → tex
│       ├── semantic-scholar.ts   /graph/v1/paper/DOI
│       └── scihub.ts        HTML parse → PDF → pdf2md
├── settings.ts              constants
└── utilities.ts             `attempt` retry helper, text clean
```

---

## Origin

Forked from `epubdomain-downloader` (v3.3.1) by Omercan Balandi. The React/Ink
TUI layer was removed and the `libgen-plus` adapter preserved. The papers
support and the POSIX-JSON output contract are new to `aaron`.

## License

[CC0 1.0 Universal](./LICENSE) — no rights reserved. Do whatever you want.
</content>
</invoke>
