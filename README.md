# aws-learning

A personal AWS knowledge base built using the [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — an LLM incrementally builds and maintains a structured wiki from raw source material, rather than re-deriving knowledge from scratch on every query.

## Structure

```text
raw/                        # source material (immutable)
  adrian-cantrill-lectures/ # Adrian Cantrill course lectures
    solution-architect/     # SAA-C03 course, grouped by topic
      <topic>/              # e.g. CloudFront/, DynamoDB/, Athena/
        videos/             # lecture recordings (git-ignored)
        transcripts/        # extracted markdown transcripts (git-ignored)
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

> **Note:** Lecture `videos/` and `transcripts/` are git-ignored because they are proprietary course material. They are kept for personal study only and are not pushed to GitHub. Only each course's `README.md` is tracked.

## Certifications

- **AWS Solutions Architect Associate** (SAA-C03)
- **AWS AI Practitioner** (AIF-C01)

## How it works

- **Raw sources** are immutable inputs — study notes, lecture transcripts, clipped articles, random thoughts
- The **wiki** is the synthesized output — the LLM reads raw sources and maintains structured, interlinked pages
- **`CLAUDE.md`** is the schema that tells the LLM how to ingest sources, answer queries, maintain cross-references, and write commit messages
- The wiki is viewed in [Obsidian](https://obsidian.md); the LLM is the editor
