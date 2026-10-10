---
name: searcharvester-extract
description: >
  Fetch a web page and extract its main content as clean markdown via
  trafilatura. Strips navigation, ads, and footers automatically. Always saves
  the FULL page to ./extracts/<id>.md and returns a pointer, so agents never
  hit output caps. With --query it additionally ranks query-relevant
  fragments of the saved page (BM25 over structure-aware chunks). Use after
  `searcharvester-search` to actually read the pages behind your URLs.
version: 1.1.0
metadata:
  hermes:
    tags: [extract, read, content, markdown, research, fragments]
    category: research
---

# Searcharvester Extract

## When to Use

- You have a specific URL and need its readable content (articles, docs, wiki pages).
- You followed up a `searcharvester-search` result and want to read it.
- You need a clean markdown version of a page for citation quoting.
- You know the sub-question the page should answer — pass it as `--query`
  and read the top fragments first instead of the whole page.

Do **not** use this skill for:

- Discovery of URLs (use `searcharvester-search` first).
- Non-HTML resources (PDFs, images) — the service expects HTML pages.

## Procedure

The extract endpoint lives at `$SEARCHARVESTER_URL` (default
`http://tavily-adapter:8000`). Use the helper script.

### Read a page (full text saved to disk)

```bash
python3 /opt/data/skills/searcharvester-extract/scripts/extract.py --url "https://en.wikipedia.org/wiki/Docker_(software)"
```

The script always fetches the complete document (paginating internally) and
writes it to `./extracts/<id>.md`. It prints a pointer, not the body:

```json
{
  "id": "abc123def4567890",
  "url": "https://...",
  "title": "Page title",
  "total_chars": 18532,
  "pages_fetched": 2,
  "path": "./extracts/abc123def4567890.md",
  "preview": "first 800 chars...",
  "hint": "Full markdown saved to ./extracts/.... Read specific parts with: ..."
}
```

Work the saved file with shell tools:

```bash
grep -ni 'keyword' ./extracts/abc123def4567890.md
sed -n '120,180p' ./extracts/abc123def4567890.md
```

### Fragment mode: read only what answers your question

```bash
python3 /opt/data/skills/searcharvester-extract/scripts/extract.py \
  --url "https://long.example.com/article" \
  --query "what you actually need from this page"
```

Same pointer, plus `fragments[]`: the saved page split into chunks that
respect headings, code fences and tables, BM25-ranked against your query.
Each fragment carries its heading path, 1-based `lines` range, neighbor chunk
ids and the text (~600 tokens each, `--fragments N` changes the count).

Escalation is cheap and ordered:

1. Neighbor chunks cover adjacent context (their `lines` are in the same file).
2. `sed -n 'a,bp' ./extracts/<id>.md` pulls any line range.
3. The full file is already on disk — no second fetch, ever.

`zero_match: true` means nothing scored: rephrase `--query` or read the file.

Re-rank a file you already saved under a new question (no fetch):

```bash
python3 /opt/data/skills/searcharvester-extract/scripts/fragments.py \
  --file ./extracts/abc123def4567890.md --query "a different angle"
```

Fragments are a read mode, not a filter — when a claim needs surrounding
qualifiers, a table, or a long code block, escalate to the file.

## Parameters reference

| Flag | Values | Default |
|---|---|---|
| `--url` | URL | required |
| `--query` | free text (your sub-question) | off — pointer only |
| `--fragments` | int | 3 (only with `--query`) |
| `--size` | ignored | kept for back-compat; the script always saves full |
| `--base-url` | adapter URL | `$SEARCHARVESTER_URL` or `http://tavily-adapter:8000` |

## Pitfalls

- **422 error**: trafilatura couldn't find extractable main-content (the page is almost entirely JavaScript or ads). Try another source from your search results — do not retry the same kind of page.
- **502 error**: the site blocked us or returned an error. Try another URL.
- **Fragment looks cut**: the fragment text is capped for output. The `lines` range still points at the complete chunk in the file — `sed` it.
- **No fragments but page exists**: `zero_match` means BM25 found no term overlap; the file still holds the full page.

## Verification

A successful response has `path` set and `total_chars` > 0. With `--query`,
`fragments[]` is non-empty unless `zero_match` is true — then rely on the
saved file.
