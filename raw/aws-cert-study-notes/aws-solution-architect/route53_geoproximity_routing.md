# Route 53 — Geoproximity Routing

## Problem / Context

You want to route users to the resource that is **physically closest** to them — and, crucially, you want the ability to **deliberately shift that boundary** so more or less traffic flows to a given resource. Geoproximity aims to provide the record that is **as close to your customers as possible**, by **distance**.

### Geoproximity vs latency

- **Latency-based** returns the record with the **lowest estimated latency** to the region.
- **Geoproximity** calculates the **physical distance** between customer and resource and answers with the **lowest distance** — _plus_ an adjustable **bias**. Same intuition, different mechanism: distance, not latency.

## Solution / Mechanics

With geoproximity you **define rules**. Each rule specifies where a resource lives, in one of two ways:

- For an **AWS resource**: the **AWS region** it's created in (e.g. `us-east-1`, `ap-southeast-2`).
- For an **external (non-AWS) resource**: **latitude & longitude coordinates**.

Either way, Route 53 knows the location of the resources and the location of the requesting customers, and directs each request to the **closest** resource.

![[R53-geoproximity-routing.png]]

In the map example there are three resources — one in **America** (blue), one in the **UK** (yellow region), and one in **Australia** (orange). Each gets an effective "area of service" shown by the tinted region around it.

### Bias

You also define a **bias** per rule. Bias adjusts how Route 53 performs the distance calculation rather than using raw physical distance (top-right: _"Routing is distance based (including bias)"_).

- A **`+` (plus) bias increases** a region's size — expanding its effective area of service so **more** surrounding traffic routes to it.
- A **`-` (minus) bias decreases** it. Increasing one region's size correspondingly **decreases neighbouring regions** (bottom-left callout).

**Worked example from the diagram:** Saudi Arabia (pink) has no local resource. It sits ~10,000 km from Australia and ~6,700 km from the UK, so by raw distance the **UK** resource would win. By applying a **large `+` bias to the Australian resource**, its effective area expands across the Middle East — so Saudi Arabian users are routed to **Australia instead of the UK**, despite Australia being physically farther.

The tinted orange overlay stretching from Asia across the Middle East illustrates the Australian region's expanded service area after the plus bias; the smaller yellow box around the UK is its (un-expanded) area.

## Exam Quick-Reference Recap

🎯 **Distance-based** routing to the closest resource — contrast with **latency-based** (lowest latency).

🎯 Rules define resource location by **AWS region** (AWS resources) or **lat/long coordinates** (external resources).

🎯 **Bias** expands or shrinks a resource's geographic service area: **`+` grows** the region (and shrinks neighbours), **`-` shrinks** it. This lets you route **more or less** traffic to a resource than pure distance would.

⚠️ Don't confuse with **geolocation** (relevance by tagged location, no distance/bias) or **latency** (estimated latency, no bias).

**Trigger keywords:** "closest resource by distance", "bias", "expand/shrink region", "route more/less traffic to a resource", "lat/long for external resources", "AWS region for resources".