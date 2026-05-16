---
title: "OpenIQ"
date: 2026-05-15
draft: false
tags: ["ai", "agents", "product-engineering", "system-design"]
description: "Building a product engineering muscle in the age of agents."
ShowToc: true
TocOpen: false
---

When coding is largely solved, what comes next?

It's been beaten to death by now, but coding has changed enough in the last six months that it's almost moot to call it coding at all. Even for devs, the job was never only code. Sometimes you're designing, mostly you're maintaining and managing the complexity of what's already there. The work usually split three ways, always unevenly: design, code, monitor. And now two of those are getting eaten.

Code, obviously. Monitoring, or observability as we are now calling it, is quieter but going the same way: agents reading logs, opening tickets, paging another on-call agent, which resolves, fixes, and rolls back breaking changes. Both of these are verifiable. Output checked against input, bounded scope. The kind of work agents are unreasonably good at. A good dev can let agents rip in this realm[^1].

Design isn't that. There's no oracle that says "this is the right product" or "this is the right shape." You can validate with users after the fact, but the up-front part (what to build, how to build it, what to ship, what to cut) is judgment. It's taste. So when the other two spokes get cheap, the part that's left is the deciding. And the deciding is mostly product engineering.

That was sort of what I wanted to try out, albeit over a few weeks. Sit a layer up the stack (plan the roadmap, design the features, write the specs) and let the agents do the rest beneath. Mostly as an exercise in taste. I called it OpenIQ. 🧭

---

## Stuck on a long flight with Claw

It started with an ad for Outreach, a CRM company, while I was waiting at the gate before an eighteen-hour flight to India. Eighteen hours holed up in a plane isn't all that bad if you're connected to Claw over WhatsApp. So after staring at the aircraft ceiling for ten minutes, I started with my first prompt: "what the hell is a CRM?" After a lot of back-and-forth, I realized the interesting part, at least for agents, was not the CRM itself but the operational memory layer on top of it: who needs attention, what context matters, and what the next safe action should be. That's how I landed on an idea I was excited to build: workflow agents for any domain. Or, if I'm roasting my own work, OpenClaw with better UI[^2]. ☠️

---

## What is OpenIQ

OpenIQ is an agentic desk assistant you can point at any small operation. Three pieces hold it up:

* an agent harness for the brain - I use Hermes Agent[^3].
* a pack of agents, skills, and MCP servers as the arms and legs - call it plugins[^4].
* a database for everything that needs to persist across runs, including the audit trail that makes the system observable.

{{< rawhtml >}}
<figure class="openiq-stack" aria-label="OpenIQ architecture stack: harness, plugins, database">
  <div class="openiq-stack__layer openiq-stack__layer--top">
    <span class="openiq-stack__name">harness</span>
    <span class="openiq-stack__role">the brain</span>
  </div>
  <div class="openiq-stack__layer openiq-stack__layer--middle">
    <span class="openiq-stack__name">plugins</span>
    <span class="openiq-stack__role">arms + legs</span>
    <span class="openiq-stack__detail">agents · skills · MCP</span>
  </div>
  <div class="openiq-stack__layer openiq-stack__layer--bottom">
    <span class="openiq-stack__name">database</span>
    <span class="openiq-stack__role">the memory</span>
  </div>
</figure>
{{< /rawhtml >}}

I started with three packs: a dental practice, a property manager, and a small law office. Different domains had different nouns, but the verbs kept repeating: notice an event, gather context, draft the next action, verify it, and put it in front of a human. The interesting part was deciding where to draw the boundary: agents for smart work, deterministic scripts for database operations. And not letting that boundary move every time I added a feature, because you don't want [agents running amok](https://x.com/DataChaz/status/2048723793120464988). 🚒

---

## A pattern forming

I started with a design doc. I would point Claw at source material, examples, docs, half-formed product references, whatever felt adjacent, and then spend a while arguing with the shape that came back. The useful part was the surface area. It could pull enough context together that I could stay in the product thought a little longer instead of dropping immediately into implementation. A lot of the early work was just that loop: ask for a shape, reject the fake parts, keep the useful bits, add more context, try again. Over time, that became the spine of `Design.md`.

Once the design was stable enough to build, I moved into a running `Tasks.md`. That became the project board: tasks, subtasks, dependencies. Nothing fancy. Just enough structure that each agent session could pick up a bounded piece of work, make a plan, implement it, verify it, and leave the board cleaner than it found it.

Each subtask became the definition of a session. The session is usually a small loop between two agents[^5], hand-knit with me in the middle, though it could probably be [Ralphed](https://ghuntley.com/loop/). The first agent would create a plan, implement it, and run the tests. The second agent would review the change: read the diff, check the plan against the result, look for missing tests or weird abstractions. Then I would start the next pass.

That loop worked, but it exposed the thing that kept getting fragile: the docs. Across sessions, project memory starts to drift. Sometimes tasks wouldn't get updated. Other times, planning a subtask would reveal a design gap and we would fail to capture it in `Design.md`. A later session would then reason from the old shape. This is not new. There is already [research showing that LLMs eventually corrupt documents during delegation](https://arxiv.org/abs/2604.15597), which claims even frontier models corrupt documents about 25% of the time. The docs do not become useless all at once. They just stop matching the work.

My workaround was to make doc updates part of the workflow. `Tasks.md` always gets updated. If a subtask changes the product or architecture, `Design.md` gets updated too. If `Design.md` changes in a major way, I revisit the invariants in `CLAUDE.md`. The goal was to keep the few project truths stable across sessions.

That was the pattern that eventually showed up over multiple sessions. The docs still drift, but treating doc upkeep as part of the workflow has at least become a mental model I now religiously follow.

---

## Anyway, here's where it ended up

OpenIQ has become the project where I'm trying to build a product engineering muscle, tinkering with ideas that keep changing how software gets built. I come across a new article or pattern, turn it into a plan, and then fold it back into the workflow. It's about 200 commits deep now, and it only gets harder as the codebase grows. But I look forward to managing the complexity (that's what devs do, after all!) and seeing how an AI-native setup performs over time.

But before you ask for receipts, not all open things are actually open. 😉 There are still obvious gaps in the agent boundary: this is no [Secure Claw](https://abhirame.github.io/posts/secure-claw/), nor does it follow the rigorous threat model I would want before releasing something public or open source. So it all stays private for now.

That said, this was an exercise in taste, and it deserves a good product video with a banger track[^6]. So here you go.

{{< rawhtml >}}
<figure class="openiq-demo-video align-center">
  <video src="/videos/openiq_demo.mp4" controls playsinline preload="metadata">
    Your browser does not support the video tag.
  </video>
</figure>
{{< /rawhtml >}}

[^1]: This is a premise, but let me be honest: not entirely true. [Thread](https://x.com/mitchellh/status/2055380239711457578).

[^2]: And funnily, Claude Legal and Claude Finance all landed in roughly the same place.

[^3]: Leaner than OpenClaw. I have a feeling OpenAI is cooking something to use OpenClaw's engine in the B2B space, so a long-term-support fork might be around the corner.

[^4]: Similar in shape to Claude Plugins. Extensibility is free, and I rely on vetted skills rather than yoloing.

[^5]: I know there is a world out there where people run multiple parallel sessions with hundreds of agents. My brain's context window is still not there yet.

[^6]: [Collect200 - Goodbye](https://www.youtube.com/watch?v=9f_LX9KXWWo).