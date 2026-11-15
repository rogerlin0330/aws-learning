# AWS Learning Wiki — Schema

This repo is an AWS learning knowledge base maintained using the LLM Wiki pattern. It covers how AWS works, architecture patterns, hands-on practice, and certification prep — in that order of priority.

## Working Agreement

- Keep `AGENTS.md` and `CLAUDE.md` identical when changing repository instructions; both contain the same schema for their respective agents.
- Start with [README.md](README.md) for repository orientation and [wiki/index.md](wiki/index.md) to locate existing coverage. Read the relevant pages and sources before editing; extend existing explanations before creating overlapping pages.
- Check `git status --short` before making changes and preserve unrelated work. A request to edit documentation does not itself request staging, committing, or pushing.

## Layers

**`raw/`** — Source material. Preserve original notes, articles, and recordings; put corrections and synthesis in the wiki. Includes:

- `raw/adrian-cantrill-lectures/solution-architect/` — Adrian Cantrill SAA-C03 course material, grouped as `<topic>/videos/` and `<topic>/transcripts/`. Both directories are git-ignored; see the tracked [course README](raw/adrian-cantrill-lectures/solution-architect/README.md). Recordings and transcripts stay local and must not be force-added to Git.
- `raw/aws-cert-study-notes/` — personal study notes, one subfolder per cert:
  - `aws-solution-architect/` — SAA-C03 notes
  - `aws-ai-practitioner/` — AIF-C01 notes
- `raw/web-clippings/` — web articles and official docs clipped as markdown
- `raw/random-thoughts/` — unstructured notes: quick insights, open questions, half-formed ideas; one file per session named `YYYY-MM-DD.md`. Once processed into the wiki, the raw file can be deleted.

Creating derived transcripts as part of requested lecture ingestion is allowed; preserve original recordings and reuse existing transcripts when suitable. Removing a processed random-thought file is optional, not a required ingest step. Changes to raw source documents otherwise require an explicit user request.

**`wiki/`** — The LLM-maintained knowledge base. The LLM creates, updates, and cross-references all pages here. You read it; the LLM writes it.

## Wiki Structure

```text
wiki/
  index.md          # catalog of pages and selected standalone diagrams
  log.md            # append-only chronological record of all operations
  overview.md       # synthesis: how everything connects; big-picture narrative
  services/         # one page per AWS service (IAM, EC2, VPC, S3, …)
  concepts/         # cross-cutting topics (networking, security, storage, …)
  patterns/         # architecture & design patterns (HA, multi-region, cost, …)
  certifications/   # exam-specific prep — a focused subset, not the main purpose
  assets/           # embedded images and editable Excalidraw diagrams
```

## Accuracy and Sources

- Treat study notes, transcripts, and existing wiki claims as material to evaluate, not automatic authority. Correct inaccurate or outdated explanations in the wiki.
- Check current AWS documentation for changing details: pricing, quotas, supported resources and Regions, defaults, feature availability, and service renames or retirements. Prefer official service documentation, API references, pricing pages, and release announcements; use upstream documentation or versioned source for SDK and library behavior.
- Cite sources near the claims they support and collect them in `## References`. Include a verification date and relevant Region, engine, or version when needed to interpret changing facts. The page's `updated` date alone does not mean every fact was reverified.
- Distinguish historical course wording from current AWS behavior. Explain the correction without presenting obsolete behavior as current exam advice. Label inference, assumptions, and unresolved uncertainty explicitly.
- Explain how a service works, its boundaries, failure modes, and design tradeoffs before exam shortcuts. Examples should state their prerequisites and whether they were run or only checked against documentation.
- Cite private lectures by course and lesson title, with a public course/lesson link when available. Do not link wiki pages to private recordings, ignored transcripts, or machine-specific paths. References may link tracked raw notes; the explanation and visual assets must remain usable without private source files.

## Page Format

Knowledge pages and `wiki/index.md` should have YAML frontmatter. `wiki/log.md` is an append-only record with its existing format; root documentation does not need wiki frontmatter.

```yaml
---
title: <Page Title>
type: service | concept | pattern | certification | overview
tags: [iam, vpc, networking, ...]
certs: [aws-saa, aws-aip]   # optional — only when relevant to exam prep
updated: YYYY-MM-DD
---
```

- Use `##` H2 sections for major topics within a page.
- Use descriptive, lowercase, hyphenated filenames for new pages. Set `updated` to the date of a substantive edit using the user's local date; do not change untouched pages just to refresh their dates.
- Cross-reference other wiki pages with markdown links: `[IAM](../services/iam.md)`.
- Exam tips go in callout blocks: `> [!tip] Exam Tip`
- Gotchas and common mistakes go in: `> [!warning] Gotcha`
- Keep a `## References` section at the bottom of knowledge pages for source links; the index and log are navigation/history exceptions.

### Images and diagrams

- Images go in `wiki/assets/` (the wiki is self-contained; do not reference images from `raw/`). Embed with descriptive alt text: `![what the diagram shows](../assets/filename.png)`. Place images immediately after the paragraph or section they illustrate.
- When raw study notes reference images (Obsidian syntax: `![[filename.png]]`), treat them as important visual aids. Copy the file from `raw/…/assets/` to `wiki/assets/` and embed it in the relevant wiki page with descriptive alt text.
- Inspect important lecture slides as well as narration. Preserve meaningful architecture, boundaries, arrows, and labels when extracting or recreating diagrams; explain historical assumptions or corrections alongside the image.
- For Excalidraw creation or edits, use the available `excalidraw-diagram` skill. Store the editable `.excalidraw` and its rendered `.png` in `wiki/assets/`, embed the preview, and link the editable source. Render and visually inspect changes before delivery.
- Reuse existing assets where appropriate. Check that filenames, image links, and diagram labels agree with the surrounding explanation.

### Cross-referencing rules

The goal is a web of knowledge, not isolated pages. Every page should link to related pages **inline** — at the first meaningful mention of a related concept in the body text, not just in References. Guiding principles:

- **Service pages** should link to the concept pages that generalize their content (e.g. `vpc.md` → `networking.md`, `iam.md` → `identity-and-access.md`)
- **Concept pages** should link back to the service pages that implement them, and to each other where they overlap
- **The overview page** should link to all major service and concept pages and explain how they connect
- **Certification pages** should link inline to every service/concept they mention
- When creating a new page, identify at minimum: which existing pages should link *to* it, and which pages it should link *to*
- Use relative Markdown links and GitHub-style heading anchors. Obsidian readers use the GFM Heading Links plugin for those anchors. When renaming a page or heading, update affected inbound links.

### Markdown conventions

All changed repository documentation and wiki Markdown must pass markdownlint (config: `.markdownlint.json`); do not rewrite immutable raw sources to satisfy lint. Key rules:

- Surround headings with blank lines (MD022)
- Surround lists with blank lines (MD032)
- Surround tables with blank lines (MD058)
- Use spaces inside table pipes: `| col | col |` not `|col|col|` (MD060)
- Always specify a language on fenced code blocks (MD040)
- No spaces inside code spans: `` `##` `` not `` `## ` `` (MD038)
- Only one `#` H1 per page — the page title (MD025; frontmatter `title:` is exempt via config)
- MD013 (line length) is disabled — prose, tables, and URLs naturally exceed 80 chars in markdown

## Operations

### Seed / Initial setup

When seeding the wiki from `raw/aws-cert-study-notes/` notes:

1. Read the relevant note file(s)
2. Create or update the corresponding wiki page(s) — synthesize, don't just copy
3. Identify cross-cutting concepts and create/update concept pages
4. Extract exam tips and add them to the appropriate `certifications/` page
5. Update `wiki/index.md` and append to `wiki/log.md`

### Ingest (course lectures)

1. Identify the requested lessons under the topic's `videos/` and `transcripts/` directories. Reuse suitable transcripts or generate missing ones locally without changing recordings.
2. Read the transcripts and inspect relevant slides. If a lesson has multiple versions, identify which updates or supplements the others rather than duplicating coverage.
3. Verify changing or questionable claims against current primary sources and share the key corrections before writing.
4. Synthesize the explanation into service, concept, and pattern pages; extract or recreate useful visuals into `wiki/assets/`. Add relevant exam tips to the certification page.
5. Add source attribution and inline cross-links, update navigation and the log, and complete the validation checklist below. Report any unavailable transcription or slide-inspection step.

### Ingest (new external source)

When the user adds a source to `raw/` and asks you to process it:

1. Read the source
2. Discuss key takeaways with the user
3. Write a summary (can be a section on an existing page or a new source summary page)
4. Update relevant service, concept, or pattern pages
5. Add exam tips to `certifications/` if applicable
6. Update `wiki/index.md` and append to `wiki/log.md`

### Ingest (random thoughts)

When the user asks you to process a file from `raw/random-thoughts/`:

1. Read the file
2. For each thought, **think critically before accepting it**:
   - Verify correctness against what you know about AWS — do not assume the thought is right just because the user wrote it
   - If a thought is **wrong or imprecise**, correct it, explain why, and file the corrected version
   - If a thought is **partially right**, refine it to the accurate version before filing
   - If a thought is **a question**, research it using existing wiki pages and your AWS knowledge; file the answer as a new section or page if valuable
   - If a thought is **genuinely insightful and correct**, synthesize it into the relevant wiki page
   - If a thought is **too vague or low-value**, skip it — not everything needs to be filed
3. Discuss non-obvious corrections or interesting findings with the user before writing
4. Append to `wiki/log.md`; no need to create a standalone summary page unless a thought warrants its own page

Apply the same cross-link, index, date, and validation rules to pages changed by random-thought ingestion.

### Query

When the user asks a question:

1. Read `wiki/index.md` to identify relevant pages
2. Read those pages
3. Synthesize an answer with citations to wiki pages
4. Verify changing or uncertain claims against primary sources as needed; distinguish newly researched information from what the wiki already says
5. If the answer is worth keeping, offer to file it as a new wiki page or section, unless the user already asked you to save it. A read-only answer does not require an index or log edit.

### Update (existing documentation)

1. Compare the affected documents with the repository and relevant sources; fix the requested inconsistency while preserving useful existing content.
2. Update related explanations, cross-links, and dates where the change affects them. Update `wiki/overview.md` when the big-picture synthesis changes and `README.md` when repository structure or reader workflows change.
3. Keep `AGENTS.md` and `CLAUDE.md` synchronized for schema changes. Update the index where its entries are affected, append one operation log entry, and validate the changed files.

### Lint

When the user asks for a health check:

- Flag contradictions between pages
- List orphan pages (no inbound links)
- Note important concepts mentioned but lacking their own page
- Suggest gaps that could be filled with a web search or new source
- Check broken page/image links, heading anchors, missing frontmatter, and missing catalog entries
- Report findings with file links; distinguish a health report from corrections actually made

## Index Maintenance Rules

`wiki/index.md` is a catalog. Format:

```text
## Services

- [IAM](services/iam.md) — Identity & access management; policies, users, roles, ARNs

## Concepts

- [Networking](concepts/networking.md) — VPC, subnets, routing, firewalls

## Patterns

- [Global & Regional AWS Architecture](patterns/global-regional-architecture.md) — Regional application tiers, resilience, and multi-region design

## Diagrams

- [VPC Gateway Endpoints](assets/vpc-gateway-endpoints.excalidraw) — Editable architecture diagram; [PNG preview](assets/vpc-gateway-endpoints.png)

## Certifications

- [AWS SAA](certifications/aws-saa.md) — Exam tips for AWS Solutions Architect Associate
```

Update the index after creating, removing, renaming, or significantly updating a page; adjust descriptions to reflect changed scope. Include selected standalone diagrams in `## Diagrams`, and keep repository guides under `## Overview`. Keep new descriptions concise and navigational; detailed explanations belong on the linked page. Do not list planned pages as though they exist or make artificial index edits for a typo fix or log-only entry.

## Log Format

`wiki/log.md` is append-only. Each entry:

```text
## [YYYY-MM-DD] <operation> | <title>

<concise description of what changed, why, and any material verification limits>
```

Operations: `seed`, `ingest`, `query`, `lint`, `update`

Append one entry per completed operation using the user's local date. Never reorder or rewrite old entries, even if their dates or operation names differ from today's convention. Use `query` when an answer is filed and `lint` for a completed health check; ordinary read-only questions need no entry.

## Validation Before Delivery

- Run markdownlint with `.markdownlint.json` on changed Markdown files. Use an installed or cached CLI when available; report if lint could not run.
- Check local page and asset targets, newly added or changed heading anchors, and inbound links affected by renames. Ensure the wiki does not depend on ignored transcripts or local video paths.
- For new pages, verify frontmatter, source attribution, inline outgoing and incoming links, the index entry, and any relevant overview or certification updates.
- Render and inspect changed diagrams; keep their editable sources and previews consistent.
- Run `git diff --check` and inspect the final diff for unrelated changes or accidentally included private material. If either instruction file changed, verify `diff -u AGENTS.md CLAUDE.md` is empty.
- Summarize the result, checks performed, and any remaining gaps. Do not imply live AWS validation when only documentation or local checks were used.

## Version Control

Repository documentation, tracked raw notes, wiki pages, and wiki assets are managed in Git. Private lecture recordings/transcripts and local application state are excluded by `.gitignore`.

When asked to generate a commit message:

1. Run `git diff --staged` to review what is actually staged — do not assume
2. If **nothing is staged**, stop and ask the user what they want to commit before proceeding
3. If changes are staged, write a commit message following the **Conventional Commits** standard:

```text
<type>(<scope>): <short imperative summary>

<body — explain what changed and why; wrap at 72 chars>
```

Common types for this repo (retain the established `doc` spelling for documentation commits):

- `doc` — adding or updating wiki pages or raw notes
- `feat` — new wiki structure, tooling, or schema changes
- `fix` — correcting wrong information in the wiki
- `chore` — repo maintenance (moving files, renaming, config)

Scope examples: `vpc`, `iam`, `ec2`, `networking`, `wiki`, `raw`, `schema`

A body is preferred whenever the diff is non-trivial — explain what knowledge was added, corrected, or restructured, not just which files changed.

Example:

```text
doc(vpc): add Regional NAT Gateway coverage

Restructured NAT GW section to distinguish connectivity type
(public/private) from availability mode (zonal/regional).
Added RNAT details: no public subnet required, auto-expands
across AZs, automatic/manual modes, port exhaustion handling,
and migration steps from zonal. Updated aws-saa.md exam notes.
```

## Certifications in Scope

- **aws-saa**: AWS Solutions Architect Associate (SAA-C03); study notes in `raw/aws-cert-study-notes/aws-solution-architect/`, exam prep in [wiki/certifications/aws-saa.md](wiki/certifications/aws-saa.md).
- **aws-aip**: AWS AI Practitioner (AIF-C01); study notes in `raw/aws-cert-study-notes/aws-ai-practitioner/`. A dedicated wiki certification page does not exist yet; create it when relevant material is ingested and update the index and README then.
