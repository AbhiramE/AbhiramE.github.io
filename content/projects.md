---
title: "Projects"
layout: "single"
url: "/projects/"
summary: "Current and past projects"
ShowToc: false
---

The current thread running through my work is simple: what does useful software look like when agents are part of the normal stack? These are the three projects I have written up so far.

## Current Projects

### A System 2 for Claw

A background knowledge layer for Claw that looks for cross-domain connections after the main response is already written. The core idea is to let the model answer from its own knowledge first, then have a quieter sub-agent search a curated graph and speak up only when it finds something genuinely additive.

- Built around a graph of 1,088 *In Our Time* episodes, using editorial cross-references, shared guests, and content links as edges.
- Uses graph traversal instead of embeddings to surface human-curated associations that keyword search would miss.
- Adds a judge step so most lookups are discarded, keeping the feature invisible unless it has a useful follow-up.

[Read the write-up](/posts/system2/)

### Secure Claw

A small, security-first Claw harness built around the idea that the agent is now the untrusted user. The project keeps the agent useful while putting a deterministic gateway between it and anything sensitive.

- Splits the system into an isolated agent container and a gateway container that owns credentials, network access, policy, and audit logs.
- Classifies tool, shell, and network actions into allow, prompt, or deny tiers with a default-deny policy.
- Routes writes through human approval and blocks obvious credential leaks or prompt-injection patterns at the boundary.

[Read the write-up](/posts/secure-claw/)

### OpenIQ

An agentic desk assistant experiment for small operations, and a way to practice product engineering in a world where agents can do more of the implementation work. The project started from a CRM rabbit hole and turned into a private system for workflow agents across domains.

- Uses a harness, plugin packs, and a database as the brain, arms, and memory of the system.
- Explores reusable operational workflows across domains like dental practices, property management, and small law offices.
- Treats design docs, task boards, reviews, and doc upkeep as part of the agent-driven development loop.

[Read the write-up](/posts/openiq/)

## Past Projects

A selection of projects from grad school and earlier work.

- **Score Predictor** — Machine learning model for predicting cricket match scores.
- **Visualizing the Evolution of Cricket** — Data visualization exploring how the sport has changed over time.
- **Recommender System** — Collaborative filtering system for recommending places to visit.
- **Internship Poller** — Automated tool for aggregating internship postings.
- **Learning to Rank with RankLib** — Information retrieval experiments using learning-to-rank algorithms.
- **Intermediate Agent for Supermarkets** — Agent-based simulation for supermarket logistics.
- **Fflaunt** — Windows Phone application.
- **SplitWork** — Android app for splitting tasks and expenses.
- **Reddit Extension** — Browser extension for enhanced Reddit browsing.
