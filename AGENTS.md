# AWS Learning Wiki — Schema

This repo is an AWS learning knowledge base maintained using the LLM Wiki pattern. It covers how AWS works, architecture patterns, hands-on practice, and certification prep — in that order of priority.

## Layers

**`raw/`** — All source documents. Immutable; the LLM reads from them but never modifies them. Includes:

- `raw/aws-cert-study-notes/` — personal study notes, one subfolder per cert:
  - `aws-solution-architect/` — SAA-C03 notes
  - `aws-ai-practitioner/` — AIF-C01 notes
- `raw/web-clippings/` — web articles and official docs clipped as markdown
- `raw/random-thoughts/` — unstructured notes: quick insights, open questions, half-formed ideas; one file per session named `YYYY-MM-DD.md`. Once processed into the wiki, the raw file can be deleted.

**`wiki/`** — The LLM-maintained knowledge base. The LLM creates, updates, and cross-references all pages here. You read it; the LLM writes it.

## Wiki Structure

```text
wiki/
  index.md          # catalog of all wiki pages — always update after any write
  log.md            # append-only chronological record of all operations
  overview.md       # synthesis: how everything connects; big-picture narrative
  services/         # one page per AWS service (IAM, EC2, VPC, S3, …)
  concepts/         # cross-cutting topics (networking, security, storage, …)
  patterns/         # architecture & design patterns (HA, multi-region, cost, …)
  certifications/   # exam-specific prep — a focused subset, not the main purpose
```

## Page Format

Every wiki page should have YAML frontmatter:

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
- Cross-reference other wiki pages with markdown links: `[IAM](../services/iam.md)`.
- Exam tips go in callout blocks: `> [!tip] Exam Tip`
- Gotchas and common mistakes go in: `> [!warning] Gotcha`
- Keep a `## References` section at the bottom for source links.
- Images go in `wiki/assets/` (the wiki is self-contained; do not reference images from `raw/`). Embed with descriptive alt text: `![what the diagram shows](../assets/filename.png)`. Place images immediately after the paragraph or section they illustrate.
- When raw study notes reference images (Obsidian syntax: `![[filename.png]]`), treat them as important visual aids. Copy the file from `raw/…/assets/` to `wiki/assets/` and embed it in the relevant wiki page with descriptive alt text.

### Cross-referencing rules

The goal is a web of knowledge, not isolated pages. Every page should link to related pages **inline** — at the first meaningful mention of a related concept in the body text, not just in References. Guiding principles:

- **Service pages** should link to the concept pages that generalize their content (e.g. `vpc.md` → `networking.md`, `iam.md` → `identity-and-access.md`)
- **Concept pages** should link back to the service pages that implement them, and to each other where they overlap
- **The overview page** should link to all major service and concept pages and explain how they connect
- **Certification pages** should link inline to every service/concept they mention
- When creating a new page, identify at minimum: which existing pages should link *to* it, and which pages it should link *to*

All wiki markdown must pass markdownlint (config: `.markdownlint.json`). Key rules:

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

### Query

When the user asks a question:

1. Read `wiki/index.md` to identify relevant pages
2. Read those pages
3. Synthesize an answer with citations to wiki pages
4. If the answer is worth keeping, offer to file it as a new wiki page or section

### Lint

When the user asks for a health check:

- Flag contradictions between pages
- List orphan pages (no inbound links)
- Note important concepts mentioned but lacking their own page
- Suggest gaps that could be filled with a web search or new source

## Index Maintenance Rules

`wiki/index.md` is a catalog. Format:

```text
## Services
- [IAM](services/iam.md) — Identity & access management; policies, users, roles, ARNs

## Concepts
- [Networking](concepts/networking.md) — VPC, subnets, routing, firewalls

## Patterns
- [HA & Multi-AZ](patterns/ha-multi-az.md) — High availability design patterns

## Certifications
- [AWS SAA](certifications/aws-saa.md) — Exam tips for AWS Solutions Architect Associate
```

Always update the index after creating or significantly updating a page.

## Log Format

`wiki/log.md` is append-only. Each entry:

```text
## [YYYY-MM-DD] <operation> | <title>
<one-line description of what was done>
```

Operations: `seed`, `ingest`, `query`, `lint`, `update`

## Version Control

The entire repo (`raw/` + `wiki/`) is managed in git. When asked to generate a commit message:

1. Run `git diff --staged` to review what is actually staged — do not assume
2. If **nothing is staged**, stop and ask the user what they want to commit before proceeding
3. If changes are staged, write a commit message following the **Conventional Commits** standard:

```text
<type>(<scope>): <short imperative summary>

<body — explain what changed and why; wrap at 72 chars>
```

Common types for this repo:

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

- **aws-saa**: AWS Solutions Architect Associate
- **aws-aip**: AWS AI Practitioner
