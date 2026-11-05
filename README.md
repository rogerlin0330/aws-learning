# aws-learning

A personal AWS knowledge base built using the [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — an LLM incrementally builds and maintains a structured wiki from raw source material, rather than re-deriving knowledge from scratch on every query.

## Structure

```text
raw/                        # source material (immutable)
  aws-cert-study-notes/     # hand-written study notes per cert
    aws-solution-architect/
    aws-ai-practitioner/
  web-clippings/            # articles and official docs clipped as markdown
  random-thoughts/          # quick insights and questions, processed into the wiki

wiki/                       # LLM-maintained knowledge base (read this)
  index.md                  # catalog of all pages
  log.md                    # append-only history of all operations
  overview.md               # big-picture synthesis
  services/                 # one page per AWS service
  concepts/                 # cross-cutting topics
  patterns/                 # architecture & design patterns
  certifications/           # exam-specific prep
  assets/                   # images embedded in wiki pages

CLAUDE.md                   # schema: conventions, workflows, and rules for the LLM
.markdownlint.json          # markdownlint config
```

## Certifications

- **AWS Solutions Architect Associate** (SAA-C03)
- **AWS AI Practitioner** (AIF-C01)

## How it works

- **Raw sources** are immutable inputs — study notes, clipped articles, random thoughts
- The **wiki** is the synthesized output — the LLM reads raw sources and maintains structured, interlinked pages
- **`CLAUDE.md`** is the schema that tells the LLM how to ingest sources, answer queries, maintain cross-references, and write commit messages
- The wiki is viewed in [Obsidian](https://obsidian.md); the LLM is the editor
