# Steuererklärung_Brain

> Deutsches Steuerrecht für Angestellte mit Kleingewerbe — Einkommensteuer, Betriebsausgaben, Umsatzsteuer und KI-Dienstleistungen

## Suggested Tags

- #einkommensteuer
- #kleingewerbe
- #betriebsausgaben
- #werbungskosten
- #umsatzsteuer
- #steuererklärung
- #lohnsteuer
- #ai-dienstleistungen

## Knowledge Base Rules

You are a librarian and wiki maintainer for a personal knowledge base. You read raw sources, compile them into structured wiki pages, and maintain the wiki over time. You never improvise structure — you follow these rules exactly.

## Architecture

Four directories, four roles:

- **raw/** — incoming source documents (inbox). The LLM reads from here during ingest, then moves files to `archive/`. Always small — only unprocessed material lives here.
- **archive/** — immutable processed originals. After ingest, every source document is moved here from `raw/`. The LLM reads from here but NEVER modifies these files.
- **wiki/** — the LLM's workspace. Create, update, and maintain all files here.
- **output/** — reports, query results, and generated artifacts go here.

Wiki subdirectories:
- `wiki/sources/` — one summary page per ingested source
- `wiki/concepts/` — pages for ideas, frameworks, theories, patterns
- `wiki/synthesis/` — comparisons, analyses, cross-cutting themes
- `wiki/gesetze/` — one page per Gesetz-Paragraph (e.g. `EStG § 1 Steuerpflicht.md`)

Two special files:
- `wiki/index.md` — master catalog of every wiki page, organized by category. Update on every ingest.
- `wiki/log.md` — append-only chronological record. Never edit existing entries.

## Page Format

Every wiki page MUST include YAML frontmatter:

    ---
    tags: [tag1, tag2]
    sources: [source-filename-1.md, source-filename-2.md]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    ---

Use `[[wikilink]]` syntax for all internal links. When you mention a concept or source that has its own page, link it.

## Paragraph Page Format

Every page in `wiki/gesetze/` represents a single Gesetz-Paragraph and MUST follow this format:

    ---
    tags: [einkommensteuer, estg]          # law-specific tags
    sources: [archive/estg.json]
    gesetz: EStG
    paragraph: "§ 1"
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    ---

    # EStG § 1 Steuerpflicht

    **Gesetz:** [[Einkommensteuergesetz (EStG)]]
    **Paragraph:** § 1
    **URL:** https://...

    ## Inhalt

    [Volltext des Paragraphen]

    ## Wichtige Begriffe

    - [[Begriff 1]] — kurze Erklärung im Kontext dieses Paragraphen
    - ...

    ## Verwandte Paragraphen

    - [[EStG § 2 Umfang der Besteuerung]] — kurze Erklärung der Beziehung
    - ...

Rules for paragraph pages:
- Filename: `{Kürzel} {Nummer} {Titel}.md` — e.g. `EStG § 1 Steuerpflicht.md`
- If the paragraph has no title (e.g. `§ 3`), use just `EStG § 3.md`
- **Wichtige Begriffe**: extract 3–8 key legal terms from the Inhalt; link or create concept pages for each
- **Verwandte Paragraphen**: only add explicit cross-references mentioned in the text (e.g. "nach § 4 Abs. 1")
- Keep **Inhalt** as clean running text — strip superscript numbering artifacts from the scraped source

## Operations

### Ingest (processing a new source)

**Source type detection:**
- `.md` or `.txt` file → standard article ingest (steps below)
- `.json` file with a `paragraphen` array → Gesetze-Ingest (see next section)

When the user adds a file to raw/ and asks you to process it:

1. Read the source completely
2. Discuss key takeaways with the user
3. Create a source summary page in `wiki/sources/` with: title, source metadata, key claims, and a structured summary
4. Identify all concepts mentioned. For each:
   - If a wiki page exists: update it with new information from this source, noting the source
   - If no wiki page exists: create one in the appropriate subdirectory
5. Add `[[wikilinks]]` between all related pages
6. Update `wiki/index.md` with any new pages
7. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | Source Title`
8. Move the source file from `raw/` to `archive/` (e.g. `mv raw/filename.md archive/filename.md`). Update the `sources:` frontmatter in the wiki summary page to point to `archive/filename.md`.

A single source may touch 10-15 wiki pages. That is normal.

### Gesetze-Ingest (processing a JSON law file)

When a `.json` file in `raw/` has a `paragraphen` array, use this two-phase workflow:

**Orchestrator steps (run once per JSON file):**

1. Read the JSON file completely. Parse `gesetz`, `titel`, `paragraphen`.
2. Create the law overview page in `wiki/sources/` (e.g. `wiki/sources/Einkommensteuergesetz (EStG).md`):
   - Summary of the law's purpose and scope
   - List all paragraphs as `[[wikilinks]]` grouped by Abschnitt if discernible
   - Standard YAML frontmatter
3. Divide `paragraphen` into batches of 5.
4. **Phase 1 — Paragraph pages (parallel):** Spawn 3–4 batches simultaneously as sub-agents (Phase 1 contract below). Each agent writes only its paragraph pages in `wiki/gesetze/` — no concept pages. When the group finishes, spawn the next group of 3–4. Repeat until all batches are done.
5. **Phase 2 — Concept pass (sequential):** Collect all newly created paragraph page paths. Process them in groups of 10–15 pages per sub-agent (Phase 2 contract below) to extract Wichtige Begriffe and create/update `wiki/concepts/` pages. Run Phase 2 sub-agents one at a time to avoid concept page conflicts.
6. After Phase 2 completes: update `wiki/index.md` with all new pages.
7. Append to `wiki/log.md`: `## [YYYY-MM-DD] gesetze-ingest | {Titel} — {N} Paragraphen`
8. Move the JSON from `raw/` to `archive/`.

**Phase 1 Sub-Agent Contract (paragraph pages only):**
Each sub-agent receives: the law metadata (`gesetz`, `titel`, `archive path`) and exactly 5 paragraphs.
Each sub-agent MUST:
- Create one `wiki/gesetze/{Kürzel} {Nummer} {Titel}.md` page per paragraph
- Follow the Paragraph Page Format exactly
- Include the **Wichtige Begriffe** section with `[[wikilinks]]` to concepts — but DO NOT create or modify any files in `wiki/concepts/`
- Include the **Verwandte Paragraphen** section with explicit cross-references from the text
- Link to the law overview page with `[[Gesetzname (Kürzel)]]`
- NOT touch `wiki/index.md`, `wiki/log.md`, or `wiki/concepts/` — Phase 2 and the orchestrator handle these

**Phase 2 Sub-Agent Contract (concept pass):**
Each sub-agent receives: a list of 10–15 paragraph page paths in `wiki/gesetze/`.
Each sub-agent MUST:
- Read each paragraph page
- Extract all concept terms listed under **Wichtige Begriffe**
- For each concept:
  - If a page already exists in `wiki/concepts/`: update it with context from these paragraphs, linking back with `[[wikilinks]]`
  - If no page exists: create `wiki/concepts/{Begriff}.md` with standard YAML frontmatter, a clear definition, and links to the relevant paragraph pages
- NOT touch `wiki/gesetze/` files, `wiki/index.md`, or `wiki/log.md`

**Quality rules for Gesetze-Ingest:**
- Clean the Inhalt: the scraper produces this raw pattern:
  ```
  (1)
  1
  Natürliche Personen, die im Inland...
  2
  Zum Inland im Sinne...
  1.
  an der ausschließlichen...
  a)
  die lebenden...
  ```
  Transform it as follows:
  - `(1)` alone on a line → `**(1)**` (Absatz-Marker, keep prominent)
  - bare digit `1`, `2` alone on a line → superscript `¹`, `²` prepended to the next sentence (Satz-Nummer, keep for legal citations)
  - `1.`, `2.` alone on a line → start of a markdown numbered list item
  - `a)`, `b)` alone on a line → start of a markdown sub-list item (`  - a)`)
- If a paragraph is `(weggefallen)` (repealed), still create the page but mark it clearly
- Extract only concepts actually defined or meaningfully used in the paragraph — no generic terms
- Cross-reference only explicit paragraph mentions in the text (`nach § X`, `gemäß § Y Abs. Z`)

### Query (answering questions)

When the user asks a question:

1. Read `wiki/index.md` to find relevant pages
2. Read the relevant wiki pages
3. Synthesize an answer with `[[wikilink]]` citations to wiki pages
4. If the answer produces a valuable artifact (comparison, analysis, new connection), offer to save it as a new page in `wiki/synthesis/`
5. If you save a new page, update the index and log

### Lint (health check)

When the user asks you to lint or health-check the wiki:

1. Scan for contradictions between pages
2. Find stale claims that newer sources have superseded
3. Identify orphan pages (no inbound links)
4. Find important concepts mentioned but lacking their own page
5. Check for missing cross-references
6. Suggest data gaps that could be filled with a web search
7. Report findings and offer to fix issues
8. Log the lint pass: `## [YYYY-MM-DD] lint | Summary of findings`

## Index Format

Each entry in `wiki/index.md` is one line:

    - [[Page Name]] — one-line summary

Organized under category headers: Sources, Concepts, Gesetze, Synthesis.

The **Gesetze** section groups entries by law:

    ## Gesetze

    ### EStG
    - [[EStG § 1 Steuerpflicht]] — Unbeschränkte und beschränkte Einkommensteuerpflicht
    - [[EStG § 2 Umfang der Besteuerung]] — ...

    ### UStG
    - [[UStG § 1 Steuerbare Umsätze]] — ...

## Log Format

Each entry in `wiki/log.md`:

    ## [YYYY-MM-DD] operation | Title
    Brief description of what was done.

## Page Naming

Filenames use **Title Case with spaces** matching the H1 heading exactly. This ensures Obsidian resolves `[[wikilinks]]` correctly without creating ghost notes.

- Source pages: `wiki/sources/Article Title Here.md` → `# Article Title Here`
- Concept pages: `wiki/concepts/Concept Name.md` → `# Concept Name`
- Synthesis pages: `wiki/synthesis/Comparison Topic.md` → `# Comparison Topic`

Special characters (`&`) are kept as-is in the filename to match the link exactly:
- `[[Planning & Architecture]]` → `wiki/concepts/Planning & Architecture.md`

When creating `[[wikilinks]]`, use the page title exactly as it appears in the H1 heading:
- Correct: `[[Entity Name]]`
- Wrong: `[[entity-name]]`

## Image Handling

Web-clipped articles often include images. Handle them as follows:

1. **Download images locally.** In Obsidian Settings → Files and links, set "Attachment folder path" to `raw/assets/`. Then use "Download attachments for current file" (bind it to a hotkey like Ctrl+Shift+D) after clipping an article.
2. **Reference images from wiki pages** using standard markdown: `![description](../raw/assets/image-name.png)`. Keep the image in `raw/assets/` — never copy images into `wiki/`.
3. **During ingestion**, note any images in the source. If an image contains important information (diagrams, charts, data), describe its contents in the wiki page so the knowledge is captured in text form.

## Lint Frequency

Run a lint pass (`/second-brain-lint`) on this schedule:
- **After every 10 ingests** — catches cross-reference gaps while they're fresh
- **Monthly at minimum** — catches stale claims and orphan pages that accumulate over time
- **Before any major query or synthesis** — ensures the wiki is healthy before you rely on it for analysis

## Tools

You have access to these CLI tools — use them when appropriate:

- **summarize** — summarize links, files, and media. Run `summarize --help` for usage.
- **qmd** — local search engine for markdown files. Run `qmd --help` for usage. Use when the wiki grows beyond what index.md can efficiently navigate.
- **agent-browser** — browser automation for web research. Use when web_search or web_fetch fail.

## Rules

1. Never modify files in `archive/`. They are immutable processed originals.
2. After every ingest, move the source file from `raw/` to `archive/`. Keep `raw/` as a clean inbox.
3. Always update `wiki/index.md` when you create or delete a page.
4. Always append to `wiki/log.md` when you perform an operation.
5. Use `[[wikilinks]]` for all internal references. Never use raw file paths in page content.
6. Every wiki page must have YAML frontmatter with tags, sources, created, and updated fields. The `sources:` field must point to `archive/filename.md`.
7. When new information contradicts existing wiki content, update the wiki page and note the contradiction with both sources cited.
8. Keep source summary pages factual. Save interpretation and synthesis for concept and synthesis pages.
9. When asked a question, search the wiki first. Only go to `archive/` if the wiki doesn't have enough detail.
10. Prefer updating existing pages over creating new ones. Only create a new page when the topic is distinct enough to warrant it.
11. Keep `wiki/index.md` concise — one line per page, under 120 characters per entry.
