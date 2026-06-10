# aaron

> **What is this?** A tiny command you run in your terminal (or hand to an AI) to find and download free books and research papers. You ask, it fetches, it prints clean JSON back to you. No browser, no account, no daemon — just a pipe between you and the world's public-domain knowledge.

---

## Wire it as a public-domain resource fetcher

**1. Install once**

```bash
git clone https://github.com/VeigaPunk/aaronplug
cd aaronplug && bun install && bun link   # puts `aaron` on your $PATH
```

**2. Fetch books**

```bash
aaron books search "moby dick"                      # search — returns JSON list
aaron books get <md5> -o ./downloads                # download one book
aaron books batch ./md5s.txt -o ./downloads         # download many
```

**3. Fetch papers**

```bash
aaron papers fetch 10.1038/nature12373              # auto-cascade: arxiv → s2 → scihub
aaron papers fetch 10.48550/arXiv.1706.03762 --mode arxiv   # force arxiv
```

**4. Wire to any AI agent** — put `aaron` on `$PATH` (done by `bun link`), then call it with any Bash/shell tool. No registration, no MCP server needed.

```bash
# Claude Code — works out of the box after bun link:
Bash(command="aaron books search 'darwin origin of species'")
Bash(command="aaron papers fetch 10.1038/nature12373")

# Codex CLI / Gemini CLI — same, they both exec shell commands directly.
# For large sci-hub outputs in Gemini, set a higher token budget:
#   "toolSettings": { "run_shell_command": { "tokenBudget": 40000 } }
```

stdout is always compact JSON. stderr is human diagnostics. `exit 0` = success, `exit 1` = `{"error":"..."}`.

---

## Full command reference

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

---

## Papers — three-tier cascade

`aaron papers fetch <DOI>` tries sources in order and returns the first that succeeds:

| order | tier     | source              | format    | best for                         |
| ----- | -------- | ------------------- | --------- | -------------------------------- |
| 1     | `scihub` | sci-hub HTML→PDF    | `markdown`| full text via `@opendocsg/pdf2md`|
| 2     | `arxiv`  | arxiv e-print tar   | `latex`   | STEM preprints (full source)     |
| 3     | `s2`     | Semantic Scholar    | `abstract`| metadata-only last resort        |

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

## Build / compile

```bash
bun run build     # emits ./build/aaron.js (node target, shebanged)
bun run compile   # emits ./standalone-executables/aaron-<platform>-<arch>
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
│   ├── adapters/            lib* mirror adapters
│   ├── data/                mirror config, document fetch, file download
│   ├── models/              Entry, DownloadResult types
│   └── papers/
│       ├── index.ts         cascade orchestrator
│       ├── arxiv.ts         DOI→arxivId→e-print tarball→tex
│       ├── semantic-scholar.ts   /graph/v1/paper/DOI
│       └── scihub.ts        HTML parse → PDF → pdf2md
├── settings.ts              constants
└── utilities.ts             `attempt` retry helper, text clean
```

Mirror list is fetched from: `https://github.com/VeigaPunk/aaronplug/blob/configuration/config.v3.json`

---

## Origin

Forked from [`epubdomain-downloader`](https://github.com/VeigaPunk/aaronplug) (v3.3.1) by Omercan Balandi. The React/Ink TUI layer was removed; the `libgen-plus` adapter preserved. Papers support and POSIX-JSON output contract are new to aaron.

## License

Unlicense — public domain. Do whatever you want.
