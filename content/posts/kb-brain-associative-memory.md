---
title: "Building a Second Brain for an AI Agent"
date: 2026-03-29
draft: true
tags: ["ai", "knowledge-graphs", "agents", "system-design"]
description: "What happens when you give an AI agent associative memory? A System 2 for slow, deliberate recall, built on a curated knowledge graph of 1,088 BBC episodes."
ShowToc: true
TocOpen: false
---

Can you augment what an LLM already knows with a curated knowledge source, and have it surface cross-domain connections the model wouldn't make on its own?

Not RAG. Not "stuff the context window with search results and hope for the best." Something closer to how associative memory actually works. You're reading about Stoicism and something clicks. You remember that thing about Confucius from last month. Nobody told you they're related. A thread just connected somewhere in the back of your mind.

I wanted to build that for an AI agent. An associative recall system that fires in the background, checks a personal knowledge base, and only speaks up when it finds something genuinely surprising. This is a follow up to a previous post about [Fireside](https://3798.substack.com/p/building-fireside), where I built a multi-agent panel discussion. That was System 1. Fast, reactive, conversational. This time I wanted to go the other direction.

---

## Kahneman, Then the arXiv Paper, Then the Idea

If you've read Kahneman's *Thinking, Fast and Slow* (or if you were paying attention during the [Fireside](https://3798.substack.com/p/building-fireside) post where Agent Kahneman showed up), you know the split. **System 1** is fast and automatic. You see 2 + 2 and the answer is just *there*. **System 2** is slow and deliberate. Long division. Tax returns. Actually reading a dense paper instead of skimming the abstract.

An LLM is almost pure System 1. You ask it something, it responds instantly from pattern-matched training data. Remarkably good at this. But there's no slow, deliberate layer checking its work against a curated source of knowledge.

This isn't just a metaphor people throw around casually anymore. Li et al. formalized it in their 2025 survey, [*"From System 1 to System 2"*](https://arxiv.org/abs/2502.17419) (225 citations and counting), which traces how modern AI is moving from reactive inference toward deliberate, multi-step reasoning. Chain-of-thought prompting, thinking modes, reflection loops. All attempts to bolt a System 2 onto what is fundamentally a System 1 architecture.

But here's the thing. Most of that work focuses on *reasoning*, making the model think harder about what it already knows. The angle I kept pulling at was different: **what if System 2 isn't about thinking harder, but about recalling better?** Not deeper reasoning over the same knowledge, but pulling in knowledge the model doesn't have. And then judging whether it's even worth mentioning.

That's the idea behind what I ended up calling **KB Brain**.

---

## The Corpus

Every system like this lives or dies by its data. I needed something broad, curated, and structured. Spanning many domains, not just one vertical.

I found it in an unlikely place. [Braggoscope](https://www.braggoscope.com/) is a fan-built index of BBC Radio 4's *In Our Time*, a show where Melvyn Bragg spends 45 minutes on a single topic with academic guests. Philosophy one week, quantum mechanics the next, then the fall of Carthage. 1,088 episodes spanning two decades.

What makes it special isn't the content though, it's the metadata. Each episode comes with academic guests, curated reading lists, Dewey Decimal classification, and (this is the important part) **editorially cross-referenced related episodes**. A human sat down and decided "Stoicism" relates to "Epicureanism" and "Cynicism," but also to "Daoism" and "Chinese Legalism." That's judgment. You can't replicate that with cosine similarity.

I scraped all 1,088 episodes and built a knowledge base from it.

---

## How It Works

The whole thing is a 4-layer system. The critical design rule up front: **System 1 responds first. The KB check happens after.** If the agent reads the KB *before* responding, it anchors on whatever it finds. The LLM's own knowledge, which is often better, gets contaminated. System 2 only adds value when it runs independently.

<!-- TODO: Insert architecture diagram here, System 1/System 2 flow -->

### The Graph

All 1,088 topics become nodes. Edges come from the editorial cross-references (weight 1.0, curated human connections), content cross-references where one topic mentions another (weight 0.5, noisier, incidental), and shared academic guests across episodes (weight 0.7). A node looks like this:

```json
{
  "stoicism": {
    "title": "Stoicism",
    "type": "topic",
    "tags": ["stoicism", "philosophy", "virtue", "zeno", "marcus-aurelius"]
  }
}
```

1,093 nodes. 8,491 edges. Average of 15.5 connections per node. No embeddings, no vector database, no NLP pipeline. Tags are just cleaned text tokens from titles and descriptions. The sophistication is in the edges, the *structure of connections*, not in the node representation.

Here's what the graph looks like around a single node:

![Knowledge graph centered on Stoicism, 23 edges. 1-hop: Epicureanism, Cynicism, Confucius. 2-hop: Comedy in Ancient Greek Theatre, The Han Synthesis, the Pelagian Controversy.](/images/system2-graph.jpeg)

Stoicism, 23 edges. The 1-hop neighborhood is what you'd expect. Epicureanism, Cynicism, Daoism. Obvious. The LLM already knows those are related. But follow the graph one more hop and you land on Comedy in Ancient Greek Theatre, The Han Synthesis, the Pelagian Controversy. *Those* are the interesting ones. Those are the connections a bare LLM won't make on its own.

### The Lookup

When the sub-agent fires, it runs a graph search. Tag match against the conversation keywords to find seed nodes, walk 1 hop out along edges (scoring neighbors by seed score × edge weight × decay), and conditionally walk a second hop if the first didn't surface enough strong hits. Rank, return the top 3–5 candidates.

Why graph traversal over embeddings? Embeddings tell you what's *semantically similar*. "Stoicism" and "Epicureanism" score high because they co-occur in similar contexts. Fine, but obvious. Graph traversal tells you what's *connected through human judgment*. A human editor decided Stoicism connects to Chinese Legalism. Two hops out, you reach Comedy in Ancient Greek Theatre, a connection that makes sense once you see it but that no embedding model would surface. The trade-off is real. "Fusion energy" won't find "nuclear power" unless the words literally appear. But for associative recall, graph structure wins.

### The Judge

This is where most systems get it wrong. They find something relevant and surface it immediately. KB Brain does the opposite. It reads the candidate summaries and asks: **does this add something the LLM didn't already say?**

Surface it if it's a historical parallel the agent missed, a surprising cross-domain connection, or a reframe that changes how you'd think about the topic. Discard it if it's something obviously related, repeats what was already covered, or amounts to "we have an entry on X" without an actual insight.

Most lookups result in nothing worth saying. That's the point. The system is designed for a ~70% discard rate. Rare, genuine "huh, I didn't think of that" moments are worth more than frequent catalog references.

### Delivery

When there's something worth saying, it arrives as a natural follow-up 15-45 seconds after the original response. When there isn't, a special token (`ANNOUNCE_SKIP`) suppresses any visible output. The user never sees the misses. The feature is invisible until it has something worth saying.

---

## Turning It On

First live session. Seven spawns:

I ask about South Korea and capitalism. The sub-agent checks the KB, finds nothing the LLM didn't already cover. Silent discard. Picasso and cubism, same story, nothing additive. Alpha Centauri, nothing. China and debt, low relevance matches, discarded.

Then I'm talking about art during the Cold War. The sub-agent fires and comes back with this: before the CIA ever thought to weaponize Abstract Expressionism, Picasso had already written the playbook with Guernica, a painting explicitly commissioned by the Spanish Republican government for the 1937 Paris Exposition to shift international opinion. The irony that Guernica was later housed at MoMA while the CIA used the same building's cultural networks for propaganda. That's an actual insight. That's the kind of connection the LLM alone doesn't make.

Except the first time it tried, it timed out. The sub-agent was mid-sentence at the 30-second mark:

> *"Before the CIA ever thought to weaponize Abstract Expressionism, Picasso had already written the playbook with Guernica, a painting explicitly commissioned by the Spanish Republican government for the 1937 Paris Exposition to shift international opinion and raise war relief funds. It"*

Cut off. Bumped the timeout to 60 seconds, re-ran, got the full insight. Classic.

Fusion energy surfaced another one, connecting China's $2.1B state-directed fusion investment to historical parallels in the corpus.

**2 insights out of 7 spawns. 29% hit rate.** The other 71% stayed completely silent. Nobody noticed. That's exactly the ratio I wanted.

---

## What I Got Wrong (So Far)

**The 30-second timeout.** Obviously. A live insight got truncated. Fixed at 60.

**Single-word tag matching.** "State-directed capitalism" gets split into three individual words ("state," "directed," "capitalism") and each one matches too broadly on its own. Multi-word concept tags would improve precision significantly.

**ANNOUNCE_SKIP.** This one's a success story that started as a failure. The first version used a `NO_INSIGHT` response, which still triggered a visible announcement in chat. "✅ Subagent finished / NO_INSIGHT" popping up after every silent discard. Seeing that in every thread was going to kill the feature. `ANNOUNCE_SKIP` makes the misses truly invisible. Small fix, essential for the whole thing to be usable.

---

## What I Learned

**Never pre-read.** The agent doesn't look at the KB before responding. This prevents anchoring bias. The LLM's training data is usually *better* than a scraped summary. The KB only adds value when it adds something *different* from what System 1 already produced.

**Make failures invisible.** A feature that announces its own failures every time will get turned off within a session. Non-negotiable.

**Background, never blocking.** The sub-agent spawns after the response is already sent. Zero latency cost. The user gets their answer instantly. If the KB has something to add, it shows up later as a natural follow-up. If it doesn't, nothing happens.

**Keep the graph dumb.** No NLP. No embeddings. No ML pipeline. The whole system runs on Python stdlib. The sophistication is in the graph edges, editorial judgment encoded as structure, not in fancy node representations.

---

## The Numbers

| | |
|---|---|
| Episodes in corpus | 1,088 |
| Graph nodes | 1,093 |
| Graph edges | 8,491 |
| Avg edges per node | 15.5 |
| Pipeline code | ~1,200 lines of Python |
| Hit rate (first session) | 29% |
| Target discard rate | ~70% |
| Sub-agent runtime | 15-45 seconds |

---

The guest edges are coded but empty. Two episodes sharing an academic expert probably share a thematic thread, but I haven't computed those yet. The corpus can grow beyond Braggoscope into articles, papers, notes. And the sub-agent pattern doesn't care where the nodes come from. It cares about the edges between them.

~1,200 lines of Python. No frameworks. No vector databases. No GPU. Just a knowledge graph, a lookup script, and a sub-agent that knows when to shut up.
