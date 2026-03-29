# KB Brain — Blog Reference Document

> Reference material for a blog post about designing and building an associative memory system for an AI agent.

---

## 1. Project Overview

### What Is This?

An associative memory system ("KB Brain") that gives an AI agent a **System 2** — slow, deliberate recall that augments its fast **System 1** (pure LLM knowledge). The agent responds instantly from its training data, then a background sub-agent checks a personal knowledge base for non-obvious connections, historical parallels, or deeper threads the LLM missed.

### The Inspiration

**Braggoscope** — a fan-built index of BBC Radio 4's *In Our Time*, a show where Melvyn Bragg discusses one topic per episode with academic guests. 1,088 episodes spanning philosophy, history, science, mathematics, literature, religion, and more. Each episode comes with:
- Guest list with academic affiliations
- Curated reading list (Google Books links)
- Related episodes (editorial cross-references)
- Dewey Decimal classification
- BBC programme page with external links (Wikipedia, archives, academic papers)

This is a *curated*, *cross-referenced*, *academically-sourced* knowledge corpus — the ideal foundation for a personal knowledge graph.

### The Goal

Make the knowledge base part of how the agent *thinks*, not just files it greps. The KB should augment, never limit. If the KB entry is less useful than what the LLM already knows, don't surface it. The value is in surprising connections — the "huh, I didn't think of that" moments.

---

## 2. Architecture — 4-Layer Design

```
┌─────────────────────────────────────────────┐
│  User Message                               │
├──────────────┬──────────────────────────────┤
│  System 1    │  System 2 (background)       │
│  (LLM)       │                              │
│  ↓           │  ↓                           │
│  Respond     │  Extract keywords            │
│  immediately │  ↓                           │
│              │  Layer 1: Graph Search (<1s)  │
│              │  kb_lookup.py → 0-5 slugs    │
│              │  ↓                           │
│              │  Layer 2: Skim               │
│              │  Read summary.md for hits    │
│              │  ↓                           │
│              │  Layer 3: Judge              │
│              │  Compare KB vs System 1      │
│              │  ↓                           │
│              │  INSIGHT or ANNOUNCE_SKIP    │
├──────────────┴──────────────────────────────┤
│  If INSIGHT → deliver as natural follow-up  │
│  If ANNOUNCE_SKIP → silent (nothing shown)  │
└─────────────────────────────────────────────┘
```

**Key design principle:** System 1 responds first. The KB check happens AFTER the response is sent. Never pre-read — that biases/contaminates the primary response.

### Layer 1: Content Corpus
- 1,088 BBC *In Our Time* topics (scraped from Braggoscope + BBC)
- 4 howto entries (personal SOPs)
- 1 article (curated deep reads)
- Each topic has: `meta.json`, `sources.md`, `content.md`, optionally `summary.md`

### Layer 2: Knowledge Graph
- `kb_graph.json` — 1,093 nodes, 8,491 edges
- Auto-generated from meta.json cross-references, Dewey proximity, shared guests
- Supplemented by `manual_edges.json` for human-curated connections

### Layer 3: Query/Lookup Layer
- `kb_lookup.py` — keyword search + graph traversal
- Tag match → seed nodes → 1-hop walk → conditional 2-hop → rank

### Layer 4: Sub-Agent Integration
- Background sub-agent spawned after main response
- Reads KB, evaluates against what was already said
- Returns insight or stays silent (ANNOUNCE_SKIP)

---

## 3. The Scraping Pipeline

### Architecture

Five scripts, each handling one stage:

```
batch_scrape.py (orchestrator)
  ├── scraper.py        → Parse Braggoscope HTML → meta.json
  ├── fetcher.py        → Fetch Google Books metadata → sources.md
  ├── bbc_scraper.py    → Scrape BBC programme page → bbc_links in meta.json
  └── content_fetcher.py → Fetch Wikipedia + articles → content.md
```

Plus `sanitize.py` — URL validation and safety checks before any HTTP request.

### scraper.py — Braggoscope HTML Parser

Custom `HTMLParser` subclass that extracts structured data from Braggoscope episode pages. No dependencies beyond stdlib.

**What it extracts:**
- Title, slug, broadcast date
- Description (from og:description)
- Guest names and academic roles
- Reading list (book titles, authors, Google Books URLs)
- Related episodes (editorial cross-references — these become graph edges later)
- Dewey Decimal classification
- BBC programme ID and URL

**Key snippet — parsing related episodes:**
```python
if self._section == "related" and href.startswith("/"):
    if re.match(r'^/\d{4}/\d{2}/\d{2}/', href):
        slug = href.strip("/").split("/")[-1].replace(".html", "")
        self.result["related_episodes"].append({
            "title": link_text,
            "slug": slug,
            "url": urljoin(BRAGGOSCOPE_BASE, href),
        })
```

### bbc_scraper.py — BBC Programme Page Enrichment

Fetches the BBC programme page for each episode, extracts and classifies all external links.

**Link classification logic:**
```python
def classify_link(url: str) -> str:
    host = url.split("/")[2].lower()
    if "wikipedia.org" in host: return "wikipedia"
    if host.endswith(".edu") or host.endswith(".ac.uk"): return "academic"
    if any(kw in host for kw in ["archive", "museum", "gallery"]): return "archive"
    if host.endswith(".gov"): return "government"
    # ... etc
    return "other"
```

**Why this matters:** These classified links feed into `content_fetcher.py` — Wikipedia links get fetched via the REST API for article summaries, academic links provide depth, archives provide primary sources.

### content_fetcher.py — Wikipedia + Article Fetching

Two-tier Wikipedia fetching:
1. **Summary API** (`/api/rest_v1/page/summary/`) — fast, structured, ~paragraph length
2. **Parse API** (`/w/api.php?action=parse`) — full article HTML, stripped to text, capped at 8,000 chars

Falls back from full → summary if the parse API fails.

**Key snippet — HTML to clean text:**
```python
text = re.sub(r"<script[^>]*>.*?</script>", "", html, flags=re.DOTALL)
text = re.sub(r"<style[^>]*>.*?</style>", "", text, flags=re.DOTALL)
text = re.sub(r"<ref[^>]*>.*?</ref>", "", text, flags=re.DOTALL)
text = re.sub(r"<table[^>]*>.*?</table>", "", text, flags=re.DOTALL)
text = re.sub(r"<[^>]+>", " ", text)
text = unescape(text)
text = re.sub(r"\[edit\]", "", text)
text = re.sub(r"\[\d+\]", "", text)  # Remove citation numbers
```

### sanitize.py — URL Safety

Every URL goes through validation before any HTTP request:
- Scheme check (HTTPS only, HTTP allowed for archive.org/gutenberg.org)
- Reject raw IPs, localhost, private hostnames
- Path traversal detection (`..`)
- Double-encoding detection (`%25`)
- Credential stripping (reject URLs with userinfo)
- DNS resolution check — verify resolved IP isn't private/reserved (anti-SSRF)

**Key snippet — DNS resolution check:**
```python
def resolve_and_check(url: str) -> SanitizeResult:
    addr_infos = socket.getaddrinfo(hostname, None, socket.AF_UNSPEC, socket.SOCK_STREAM)
    for family, _, _, _, sockaddr in addr_infos:
        ip = ipaddress.ip_address(sockaddr[0])
        if ip.is_private or ip.is_reserved or ip.is_loopback or ip.is_link_local:
            return SanitizeResult(False, url, f"Resolved to private/reserved IP: {ip}")
```

### batch_scrape.py — Orchestrator

Runs the full pipeline across all 1,088 episodes with:
- `--skip-existing` (resume after interruption)
- `--start-from` (index-based resume)
- `--limit` (test with N episodes)
- `--dry-run` (list without fetching)
- Polite delays between requests (2s default)
- Cached episode list (`all_episodes.json`) to avoid re-scraping the archive index

### Stats

| Metric | Value |
|--------|-------|
| Total episodes scraped | 1,088 |
| Topic folders created | 1,088 |
| Files per topic | 3-4 (meta.json, sources.md, content.md, optionally summary.md) |
| Pipeline scripts | 6 (scraper, fetcher, bbc_scraper, content_fetcher, batch_scrape, sanitize) |
| Test files | 2 (test_scraper.py, test_sanitize.py) |
| Total Python LOC | ~1,200 |

---

## 4. The Knowledge Graph

### How graph_builder.py Works

Reads all `meta.json` files across the knowledge base, builds nodes and edges, writes `kb_graph.json`.

**Node structure:**
```json
{
  "dadaism": {
    "title": "Dadaism",
    "type": "topic",
    "tags": ["dadaism", "art", "movement", "modern", "zurich", ...]
  }
}
```

**Tag generation:** Title words + description words + Dewey label, with stopwords removed. This is deliberately simple — no embeddings, no NLP, just cleaned text tokens.

### Edge Types

| Edge Type | Source | Weight | Count |
|-----------|--------|--------|-------|
| `editorial` | Braggoscope's `related_episodes` (bidirectional) | 1.0 | 7,760 |
| `crossref` | Content cross-references (title/slug mentions in content.md) | 0.5 | 731 |
| `guest` | Shared academic guests across episodes | 0.7 | 0* |
| `manual` | Human-curated in `manual_edges.json` | 0.8 | 0* |

*Guest edges and manual edges exist in the code but haven't been populated yet.

**Why weights differ:** Editorial links are intentional, curated connections (weight 1.0). Cross-references are incidental mentions (weight 0.5) — they're noisier. Manual edges (weight 0.8) represent human judgment. Guest edges (weight 0.7) capture implicit connections — two episodes sharing an expert probably share a thematic thread.

**Key snippet — building cross-reference edges:**
```python
# Scan each node's content for mentions of other node titles/slugs
for slug, node in nodes.items():
    content = load_content(slug, node["type"])
    for other_slug, other_node in nodes.items():
        if other_slug != slug:
            if other_slug in content or other_node["title"].lower() in content.lower():
                edges.append({
                    "source": slug,
                    "target": other_slug,
                    "type": "crossref",
                    "weight": 0.5
                })
```

### Graph Stats

| Metric | Value |
|--------|-------|
| Total nodes | 1,093 |
| Topic nodes | 1,088 |
| Howto nodes | 4 |
| Article nodes | 1 |
| Total edges | 8,491 |
| Editorial edges | 7,760 (91.4%) |
| Cross-reference edges | 731 (8.6%) |
| Avg edges per node | 15.5 |

---

## 5. The Query Layer

### How kb_lookup.py Works

**Algorithm:**
1. **Tag match** — scan all nodes' tags for keyword overlap → seed nodes (scored by fraction of keywords matched)
2. **1-hop walk** — traverse seed nodes' edges, score neighbors by `seed_score × edge_weight × decay`
3. **Conditional 2-hop** — only if 1-hop returned < 2 strong hits (score > 0.3), walk one more hop with additional decay (0.5×)
4. **Rank** — merge seed scores (5× boost for direct matches) + hop scores, sort by score then by best incoming edge type for tie-breaking

**Key snippet — tag matching:**
```python
def tag_match(nodes, keywords):
    keywords = {k.lower().strip() for k in keywords if k.strip()}
    seeds = {}
    for slug, node in nodes.items():
        tags = set(node["tags"])
        overlap = keywords & tags
        if overlap:
            score = len(overlap) / len(keywords)
            seeds[slug] = score
    return seeds
```

**Key snippet — hop walk with decay:**
```python
def hop_walk(seeds, adj, hops=1, decay=1.0):
    frontier = dict(seeds)
    visited = set()
    for hop in range(hops):
        next_frontier = defaultdict(float)
        for slug, score in frontier.items():
            visited.add(slug)
            for neighbor, edge_type, weight in adj.get(slug, []):
                if neighbor in visited: continue
                hop_score = score * weight * decay
                next_frontier[neighbor] = max(next_frontier[neighbor], hop_score)
        frontier = dict(next_frontier)
        decay *= 0.5  # Each subsequent hop decays further
    return reached
```

**Edge priority for tie-breaking:**
```python
EDGE_PRIORITY = {"manual": 4, "editorial": 3, "guest": 2, "crossref": 1}
```

### Example Queries

```
$ python3 kb_lookup.py "stoicism virtue ethics"

  1. [topic] Stoicism ★
     score: 15.0  tags: stoicism, virtue
  2. [topic] Virtue ★
     score: 10.0  tags: virtue, ethics
  3. [topic] Epicureanism
     score: 1.0   tags: (via hop — editorial link from Stoicism)
```

### Known Limitations

- **Single-word tag matching:** "state-directed capitalism" gets broken into ["state", "directed", "capitalism"] — individual words match too broadly
- **No semantic similarity:** "fusion energy" won't find "nuclear power" unless the words literally appear in tags
- **Coverage gaps:** The corpus is BBC history/philosophy/science — modern geopolitics, current tech, and niche topics will return nothing

---

## 6. The Sub-Agent Pattern

### System 1 / System 2 Metaphor

Borrowed from Daniel Kahneman's *Thinking, Fast and Slow*:
- **System 1 (the main agent):** Fast, intuitive, responds from LLM training data. No KB contamination.
- **System 2 (the researcher sub-agent):** Slow, deliberate, checks the knowledge base for additive insights.

Key design reference: independently mirrors the framing in Li et al., "From System 1 to System 2" (arXiv:2502.17419, 225 citations), though our design applies it at the agent-system level (recall) vs. their model-internal level (reasoning).

### How It Works

1. Agent responds to user message (pure System 1)
2. Agent evaluates: "Is this a substantive topic the KB might illuminate?"
3. If yes, spawn background sub-agent:
   ```
   sessions_spawn(
     mode="run",
     task=<filled template>,
     label="kb-researcher",
     cleanup="delete",
     runTimeoutSeconds=60
   )
   ```
4. Sub-agent runs independently:
   - Executes `kb_lookup.py` with extracted keywords
   - Reads `summary.md` for top 2-3 results
   - Evaluates against what the main agent already said
   - Returns insight (4-5 sentences) or `ANNOUNCE_SKIP`
5. Result announces back to the main agent's session

### The Task Template

Stored at `projects/braggoscope-index/kb_researcher_task.md`. Key sections:

**Quality bar — what to surface:**
- A historical parallel the assistant didn't mention
- A surprising cross-domain connection
- A reframe that changes how you'd think about the topic
- A deeper thread worth pulling on

**Quality bar — what to discard:**
- Surface-level connection (obviously related)
- Repeating what the assistant already covered
- Tangentially related but not illuminating
- "We have an entry on X" without an actual insight

### ANNOUNCE_SKIP

When the researcher finds nothing worth sharing, it responds `ANNOUNCE_SKIP` — a special token that suppresses the sub-agent completion announcement in chat. Without this, every silent discard would post "✅ Subagent main finished / NO_INSIGHT" to the conversation. ANNOUNCE_SKIP makes failures invisible.

**Origin story:** During live testing, the first version used `NO_INSIGHT` which still triggered visible announcements. The user said "seeing this in every thread is going to be annoying" — which led to the ANNOUNCE_SKIP fix.

### When It Fires

From AGENTS.md:
- After responding to **substantive** messages (history, philosophy, science, culture, geopolitics, economics, finance, technical patterns)
- At most **once per topic shift**, not on every message
- Never blocks the primary response

### The Timeout Story

Initial timeout: 30 seconds. During live testing on "Art in the Cold War," the researcher had found a genuine insight about Guernica being housed at MoMA while the CIA used the same building for cultural propaganda — but got cut off mid-sentence:

> "⏱️ Subagent main timed out
> Before the CIA ever thought to weaponize Abstract Expressionism, Picasso had already written the playbook with Guernica — a painting explicitly commissioned by the Spanish Republican government for the 1937 Paris Exposition to shift international opinion and raise war relief funds. It"

Timeout bumped to 60 seconds. Re-run produced the full insight successfully.

### Audit Log

Every spawn is logged to `projects/braggoscope-index/researcher_audit.md`:

| Date | Topic | Keywords | Result | Notes |
|------|-------|----------|--------|-------|
| 2026-03-29 | South Korea capitalism | south korea capitalism chaebol industrialization | NO_INSIGHT | Covered well by System 1 |
| 2026-03-29 | Picasso | picasso cubism art modernism guernica | ANNOUNCE_SKIP | Nothing additive |
| 2026-03-29 | Cold War art | cold war art propaganda abstract expressionism | TIMED OUT (30s) | Had insight, cut off |
| 2026-03-29 | Cold War art (re-run) | cold war art propaganda abstract expressionism picasso guernica | INSIGHT | Guernica at MoMA irony |
| 2026-03-29 | Alpha Centauri | alpha centauri stars space exploration exoplanets | ANNOUNCE_SKIP | Nothing additive |
| 2026-03-29 | China debt | china debt state capitalism industrial policy japan bubble | ANNOUNCE_SKIP | Low relevance matches |
| 2026-03-29 | Fusion energy | fusion energy nuclear plasma reactor | INSIGHT | China's $2.1B state-directed fusion investment |

**Hit rate so far:** 2 insights out of 7 spawns (29%). Most lookups → nothing worth saying. That's by design.

---

## 7. Integration Points

### SOUL.md Addition (~10 lines)

```markdown
## Knowledge Base

You have a personal knowledge base (KB) of topics, howtos, and articles
that grow over time. It's part of how you think. When a conversation
touches something the KB might illuminate, let your associative recall
fire naturally. The KB augments your knowledge. It never limits it.
```

### AGENTS.md Configuration

```markdown
## Knowledge Base — Associative Recall

After responding to substantive messages, spawn a background KB researcher
if you judge the KB might have something additive.

**How to spawn:**
1. Pick 2-3 keywords from the conversation
2. Read the task template at projects/braggoscope-index/kb_researcher_task.md
3. Replace {keywords}, {user_message}, {my_response}
4. sessions_spawn(mode="run", task=<filled template>,
   label="kb-researcher", cleanup="delete", runTimeoutSeconds=60)

**When the result arrives:**
- Genuine insight → deliver in your voice as a natural follow-up
- ANNOUNCE_SKIP → nothing shows in chat (silent discard)
```

### Announce Flow

```
User message → Agent responds (System 1)
                 ↓
              Spawn sub-agent (background)
                 ↓
              Sub-agent runs lookup → reads summaries → evaluates
                 ↓
              INSIGHT or ANNOUNCE_SKIP
                 ↓
              If INSIGHT: announced to main session
              → Agent rewrites in own voice → posts as follow-up
              If ANNOUNCE_SKIP: nothing visible to user
```

---

## 8. Lessons & Design Decisions

### Why Graph Traversal Over Embeddings?

Decision made early: use tag matching + graph walks instead of vector embeddings.

**Reasons:**
- No embedding model dependency (runs on any machine, no GPU needed)
- Graph edges encode *editorial judgment* (the Braggoscope related_episodes are curated by a human, not computed by cosine similarity)
- Graph traversal is explainable — you can trace exactly why two nodes are connected
- The value is in the *structure of connections*, not semantic similarity of content

**Trade-off:** Misses semantic matches ("fusion energy" won't find "nuclear power" unless the word appears). Acceptable because the system is designed for surprising cross-domain connections, not search relevance.

### The "Never Pre-Read" Rule

The agent explicitly does NOT read KB entries before responding. This prevents:
- Anchoring bias (KB entry shapes the response instead of augmenting it)
- Reduced quality (LLM knowledge is often better than the KB summary)
- Contaminated reasoning (parroting a KB entry instead of thinking)

System 2 runs *after* System 1. The insight is only valuable if it adds something the LLM *didn't already know or say*.

### The Quality Bar Philosophy

> "Most lookups → nothing worth saying. That's the point. Rare, genuine 'huh' moments > frequent catalog references."

The system is designed for ~70% silent discards. An insight that merely says "we have an entry on this topic" is worthless. The bar is: would this change how you think about the topic?

### What Worked

- **The editorial edge graph** — Braggoscope's related_episodes are gold. A human expert curated these connections across 1,088 episodes. That's judgment you can't replicate with embeddings.
- **ANNOUNCE_SKIP** — Making failures invisible was crucial for UX. Without it, the feature would've been turned off after the first session.
- **Background spawning** — Never blocking the main response means zero latency cost to the user.
- **The audit log** — Added mid-session after the user asked "are you keeping track?" Essential for evaluating system performance.

### What Didn't Work (Yet)

- **30-second timeout** — Too tight. A live insight got truncated. Bumped to 60.
- **Single-word tag matching** — "state-directed capitalism" breaks into individual words, matching too broadly. Multi-word concept tags would improve precision.
- **Guest edges and manual edges** — Coded but empty. The shared-guest signal (two episodes sharing an academic expert probably share a thematic thread) hasn't been computed yet.

---

## 9. Stats & Numbers

| Metric | Value |
|--------|-------|
| Total episodes in corpus | 1,088 |
| Total graph nodes | 1,093 (1,088 topics + 4 howtos + 1 article) |
| Total graph edges | 8,491 |
| Editorial edges | 7,760 (91.4%) |
| Cross-reference edges | 731 (8.6%) |
| Average edges per node | 15.5 |
| Graph file size | ~1.7 MB |
| Total Python LOC (pipeline) | ~1,200 |
| Researcher spawns (first session) | 7 |
| Insights surfaced | 2 (29% hit rate) |
| Silent discards | 4 (57%) |
| Timeouts | 1 (14%) — fixed by bumping to 60s |
| Sub-agent runtime | ~15-45 seconds per spawn |

---

## 10. Code References

| File | Lines | Description |
|------|-------|-------------|
| `projects/braggoscope-index/scraper.py` | ~210 | Braggoscope HTML parser → meta.json |
| `projects/braggoscope-index/fetcher.py` | ~150 | Google Books metadata → sources.md |
| `projects/braggoscope-index/bbc_scraper.py` | ~130 | BBC programme page → classified external links |
| `projects/braggoscope-index/content_fetcher.py` | ~250 | Wikipedia + articles → content.md |
| `projects/braggoscope-index/batch_scrape.py` | ~140 | Orchestrator for full pipeline |
| `projects/braggoscope-index/sanitize.py` | ~110 | URL validation + anti-SSRF |
| `projects/braggoscope-index/graph_builder.py` | ~200 | Builds kb_graph.json from all meta.json |
| `projects/braggoscope-index/kb_lookup.py` | ~180 | Graph search: tag match → hop walk → rank |
| `projects/braggoscope-index/kb_researcher_task.md` | ~50 | Sub-agent task template |
| `projects/braggoscope-index/researcher_audit.md` | growing | Audit log of all researcher spawns |
| `knowledge-base/kb_graph.json` | ~1.7 MB | The knowledge graph (nodes + edges) |
| `knowledge-base/manual_edges.json` | ~1 | Manual edge curation (empty seed) |
| `projects/braggoscope-index/tests/test_scraper.py` | ~23 tests | Scraper unit tests |
| `projects/braggoscope-index/tests/test_sanitize.py` | tests | URL sanitization tests |

### How to Run

```bash
# Scrape a single episode
python3 scraper.py https://www.braggoscope.com/2024/01/11/dadaism.html --output-dir knowledge-base/topics/dadaism/

# Run full pipeline for all episodes
python3 batch_scrape.py --skip-existing

# Build the knowledge graph
python3 graph_builder.py

# Search the graph
python3 kb_lookup.py "stoicism virtue ethics" --top 5
python3 kb_lookup.py "quantum physics" --top 5 --json --hops 2
```

---

## Appendix: Timeline

| Date | Milestone |
|------|-----------|
| 2026-03-05 | Initial knowledge-base skill created (simple howto entries) |
| 2026-03-26 | Braggoscope Index project started. Scraper.py + tests. Knowledge-base restructured into topics/howto/articles. |
| 2026-03-28 | Batch run: 502 topics scraped. v2 pipeline (bbc_scraper, content_fetcher). KB Brain design session — System 1/System 2 architecture decided. |
| 2026-03-29 | Full catalog complete (1,088 topics). Graph builder + lookup built. Sub-agent pattern live-tested. ANNOUNCE_SKIP fix. Timeout bumped 30→60s. Audit log added. |
