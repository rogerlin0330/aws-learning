# Route 53 — Geolocation Routing

## Problem / Context

You want resolution decisions driven by **the location of the user and the location of your resources** — for example to **restrict content** to a market, serve **language-specific content**, or **load balance across regional endpoints** based on customer location.

In many ways it resembles latency routing, but instead of latency it uses **location** to influence decisions.

⚠️ **The single most important point (stressed repeatedly for the exam):** geolocation **does not return the closest record — it only returns _relevant_ (location-matching) records.** You will _not_ get a Canadian record if you're in the UK and no closer record exists.

## Solution / Mechanics

When you create records, you **tag each with a location**. There are four tag types, from most specific to least:

1. **Subdivision** — a **US state** (the smallest/most specific type).
2. **Country** — using ISO standard country codes.
3. **Continent** — using ISO continent codes (e.g. `SA` for South America).
4. **Default** — an optional catch-all.

![[R53-geolocation-routing.png]]

At resolution time (right-hand callouts): an **IP check verifies the location of the user** — depending on the DNS setup this may be the user directly or the **resolver**, though in most cases they represent the same location.

### Matching order (the core algorithm)

The left-hand callout lists the order. Route 53 takes the user's location and checks for matching records, **stopping at the first level that produces a match**:

1. **State** — if the user is in the US, check for records tagged with the user's state. Any match → returned, **process stops**.
2. **Country** — if no state match, check for records tagged with the user's country. Match → returned, stop.
3. **Continent** — if no country match, check the user's continent. Match → returned, stop.
4. **Default** — an optional catch-all returned when no location-specific record is relevant.

It returns the **most specific** relevant record. If **nothing matches and there is no default record**, Route 53 returns **"NO ANSWER"**.

So the outcome is always one of: a relevant (matching) record, the default record, or NO ANSWER — **never** a "nearest" record chosen by proximity.

## Use Cases

- **Regional restrictions** — e.g. create a US record so only US-located users receive it (content for the US market only).
- **Language-specific content.**
- **Load balancing across regional endpoints** based on customer location.

## Exam Quick-Reference Recap

🎯 **Relevance, not proximity.** Geolocation returns location-matching records only — never the geographically closest as a fallback.

🎯 **Specificity order:** State (US subdivision) → Country → Continent → Default. Most specific match wins; process stops at the first match.

🎯 If no match and no default → **NO ANSWER**.

🎯 **Use for:** regional restriction, language content, regional load balancing.

⚠️ Don't confuse with geoproximity — geolocation has **no concept of distance or bias**; a user in a US state with no matching state record falls through to country, then continent, then default.

**Trigger keywords:** "restrict content by region", "block/allow by country", "language-specific content", "state / country / continent", "default record", "NO ANSWER", "not closest — only relevant".