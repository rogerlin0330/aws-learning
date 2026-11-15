# aws-learning

A personal AWS knowledge base built using the [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — an LLM incrementally builds and maintains a structured wiki from raw source material, rather than re-deriving knowledge from scratch on every query.

The priority is understanding how AWS works, architecture patterns, hands-on practice, and then certification prep.

## Start here

- [Wiki index](wiki/index.md) — the complete catalog of services, concepts, patterns, diagrams, and certification prep
- [Overview](wiki/overview.md) — how the services and architectural ideas connect
- [Operation log](wiki/log.md) — what has been added, corrected, or expanded

Read the Markdown directly on GitHub or open the repository as an Obsidian vault. Section links use GitHub-style heading anchors; in Obsidian, use the GFM Heading Links plugin to follow them.

## What's covered

- **Identity, security, and governance:** [IAM](wiki/services/iam.md), [Organizations](wiki/services/organizations.md), [Secrets Manager](wiki/services/secrets-manager.md), [WAF](wiki/services/waf.md), [Shield](wiki/services/shield.md), [Config](wiki/services/config.md), and [GuardDuty](wiki/services/guardduty.md)
- **Networking and edge delivery:** [VPC](wiki/services/vpc.md), [DNS](wiki/concepts/dns.md), [Route 53](wiki/services/route53.md), [CloudFront](wiki/services/cloudfront.md), and [private service access](wiki/services/vpc-interface-endpoints.md)
- **Compute and application architecture:** [EC2](wiki/services/ec2.md), [containers](wiki/concepts/containers.md), [Kubernetes](wiki/concepts/kubernetes.md), [Lambda](wiki/services/lambda.md), [API Gateway](wiki/services/api-gateway.md), and [serverless architecture](wiki/patterns/serverless-architecture.md)
- **Storage, databases, and analytics:** [storage fundamentals](wiki/concepts/storage.md), [S3](wiki/services/s3.md), [RDS](wiki/services/rds.md), [DynamoDB](wiki/services/dynamodb.md), [ElastiCache](wiki/services/elasticache.md), [Neptune](wiki/services/neptune.md), [OpenSearch](wiki/services/opensearch.md), and [data integration](wiki/concepts/data-integration.md)
- **Messaging and design tradeoffs:** [streaming versus messaging](wiki/concepts/streaming-vs-messaging.md), [architecture evolution](wiki/patterns/architecture-evolution.md), [global and regional architecture](wiki/patterns/global-regional-architecture.md), [scaling](wiki/patterns/scaling.md), and [cost optimization](wiki/patterns/cost-optimization.md)

## Structure

```text
raw/                        # source material (immutable)
  adrian-cantrill-lectures/  # Adrian Cantrill course lectures
    solution-architect/     # SAA-C03 course, grouped by topic
      <topic>/              # e.g. CloudFront/, DynamoDB/, Athena/
        videos/             # lecture recordings (git-ignored)
        transcripts/        # extracted markdown transcripts (git-ignored)
  aws-cert-study-notes/      # hand-written study notes per cert
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
  assets/                   # images and editable Excalidraw diagrams

AGENTS.md                   # schema: conventions, workflows, and rules for agents
CLAUDE.md                   # corresponding instructions for Claude
.markdownlint.json          # markdownlint config
```

> **Note:** Lecture `videos/` and `transcripts/` are git-ignored because they are proprietary course material. They are kept for personal study only and are not pushed to GitHub. The tracked [course README](raw/adrian-cantrill-lectures/solution-architect/README.md) explains their organization; synthesized explanations and diagrams live in the self-contained wiki.

## Certifications

- **AWS Solutions Architect Associate** (SAA-C03) — [wiki exam prep](wiki/certifications/aws-saa.md) and [raw study notes](raw/aws-cert-study-notes/aws-solution-architect/)
- **AWS AI Practitioner** (AIF-C01) — [raw study notes](raw/aws-cert-study-notes/aws-ai-practitioner/); a dedicated wiki certification page has not been created yet

## How it works

- **Ingest sources:** the LLM reads study notes, lecture transcripts, clipped articles, or random thoughts, checks claims, and synthesizes them into linked service, concept, and pattern pages. Exam tips go into certification prep.
- **Preserve source material:** raw sources remain unchanged. Processed random-thought files may be deleted after their useful content has been filed.
- **Ask questions:** the LLM starts with the wiki index, reads relevant pages, and answers with links to the supporting pages.
- **Maintain the wiki:** significant page updates are reflected in the index and appended to the operation log. Pages use YAML frontmatter, inline cross-references, source references, and local assets; Markdown follows [.markdownlint.json](.markdownlint.json).

[AGENTS.md](AGENTS.md) and [CLAUDE.md](CLAUDE.md) define the detailed conventions and workflows. The wiki is read in [Obsidian](https://obsidian.md) or on GitHub; the LLM is the editor.
