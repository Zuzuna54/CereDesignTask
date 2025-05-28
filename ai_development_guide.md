# A Responsible Developer's Guide to Generative AI — My Day-to-Day Playbook

Think of this as me walking you through my desk, showing you which buttons I push, why I push them, and what guard-rails keep me (and the codebase) out of trouble. I'll split the narrative into the two moments where generative AI helps me most: (1) shaping system-level diagrams, and (2) cranking out real code. At every turn I'll flag the upside, the downside, and the small habits that turn "AI magic" into reliable engineering practice.

## 1. Architectural & Data-Flow Diagrams

### Why I reach for AI here

- **Velocity**: A five-minute AI sketch in Mermaid beats thirty minutes of box-dragging in Miro.
- **Exploration**: I can riff through "what-ifs" (event-driven vs. REST, star vs. mesh, etc.) without redrawing from scratch.
- **Documentation**: The textual → visual workflow means the diagram is the doc — no chance they diverge.

### Tools on my bench

#### Mermaid's built-in AI generator

- **Platform**: [mermaid.js.org](https://mermaid.js.org) — free account
- **Use case**: Perfect for a quick "show me a service mesh" boost directly in the browser

#### Cursor / Windsurf with @docs crawler

- **Setup**: Pointed at the repo for context
- **Model preference**: DeepSeek-R1 as the underlying model; it "groks" YAML-heavy infra configs better than most

#### Miro + Mermaid extension

- **Important**: I never drag raw SVGs into Miro — the text labels disappear because Miro's renderer treats `<text>` differently
- **Best practice**: Instead I paste the Mermaid source into the extension so Miro renders the SVG itself
- **Note**: The geometry may shift compared to VS Code or GitHub's renderer, but the text is intact and everyone can read it

### Guard-rails I've built

#### 1. Prompt like a playwright, not a telegram operator

I declare every entity plus its relationships in words before I ever ask for code:

```
"Create a flowchart with these actors:
– User-Portal
– API-Gateway
– Auth-Service
– Billing-Service
Connect them so that…"
```

#### 2. Iterate in child-steps

After each AI-generated edit I read the source, render it, and confirm intentions line-by-line. No next step until the current diagram is 100% clear.

#### 3. Take running notes

These become ready-made talking points when I later write the design doc (Gemini 2.5 Pro is my drafting buddy for prose).

### Pros, cons & trade-offs

| Aspect         | Using AI                                                                                  | Skipping AI                                                                             |
| -------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Pros**       | • Speed: First draft in minutes<br>• Consistency: Textual DSL keeps version control clean | • Deep manual understanding<br>• Full renderer control: No surprise line-breaks in Miro |
| **Cons**       | • Risk of shallow grasp: If I paste prompts blindly, I miss subtle integration edges      | • Time sink: Complex topologies take ages to hand-draw                                  |
| **Mitigation** | Enforce the "describe-before-generate, review-after-generate" ritual                      | —                                                                                       |

## 2. AI-Assisted Coding

### The runway before I ever hit "Generate"

1. **Write an Implementation Plan**

   - Concrete roadmap, atomic tasks, files to touch, and expected outcomes

2. **Configure project rule-sets**

   - Cursor's YAML lives under `.cursor/config`
   - This tells the model which layers may import which, lint rules, naming conventions, etc.

3. **Index the codebase regularly**
   - Context equals quality — no index, no clue

### My tool + model stack

| Layer            | Choice           | Why                                                               |
| ---------------- | ---------------- | ----------------------------------------------------------------- |
| **Editor agent** | Cursor           | Cheaper & faster than Windsurf; strong rule-set features          |
| **Large model**  | Claude 3.7 / 4.0 | Big context window for whole-file diffs                           |
| **Fast model**   | Gemini 2.5       | Snappy for boilerplate, though outages & truncated replies happen |

### The on-ramp: micro-steps, micro-reviews

1. **Generate the smallest diff that compiles**

   - No 400-line deltas

2. **Read every token the AI writes**

   - Yes, even the boilerplate. The moment comprehension stops, I slow down

3. **Manual for mission-critical logic**
   - Business rules, crypto, anything regulatory — my hands on the keyboard
   - AI can stub, I flesh out

### Pros, cons & trade-offs

| Aspect         | Using AI                                                                                                                                      | Not using AI                                                   |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **Pros**       | • Throughput: CRUD endpoints, tests, even Terraform skeletons appear in seconds<br>• Consistency: Linter rules & templates applied flawlessly | • Mastery: I own every line<br>• Fewer hidden "hallucinations" |
| **Cons**       | • Diluted developer intuition: Easy to lose touch with call-graphs<br>• Over-trust in unseen code paths                                       | • Slower to MVP<br>• Tedious boilerplate                       |
| **Mitigation** | Force micro-steps, run static analysis, and schedule manual code-reading sessions                                                             | —                                                              |

## 3. Prompt & Context Hygiene — The Secret Sauce

**Remember**: Every call to an AI API is stateless.

ChatGPT, Cursor, Windsurf all bolt extra plumbing on top: they bundle previous turns, repo snippets, or @docs results into each request. Knowing this means I can spot when context is missing and supply it explicitly.

### Prompt anatomy (my template)

1. **Role**: "You are a TypeScript backend engineer…"
2. **Goal**: "Implement a NestJS service for user onboarding…"
3. **Constraints**: "Must follow Hexagonal Architecture, use Zod for validation…"
4. **Context**: Inline code snippets, schema.prisma excerpt, etc.
5. **Step size**: "Produce only the Service class with interfaces; no controller."

### Best practices

- **Chunk big asks**: If I notice the prompt creeping above 300 lines, I slice it and feed the chunks sequentially
- **Echo critical context back**: I ask the model to restate the requirement before coding — a quick sanity echo that costs 10 seconds and saves 30 minutes

## 4. Balancing Understanding vs. Speed — My Personal Ruleset

| Scenario                   | Approach                                                                                            |
| -------------------------- | --------------------------------------------------------------------------------------------------- |
| **Green-field prototypes** | Bias toward AI for speed, then refactor manually once architecture stabilizes                       |
| **Core domain logic**      | Hand-code, but let AI draft tests and docs                                                          |
| **On-call hot-fixes**      | Never ship AI code unreviewed. Sleep later, read now                                                |
| **Learning moments**       | Purposefully disable autocomplete, write raw code, then compare with AI output to sharpen instincts |

## 5. Closing Thoughts

Generative AI is the **power tool, not the craftsman**. Used blindly it builds rickety houses; used deliberately it frees me to design better structures and focus on edge-cases humans actually care about.

The rituals above — precise prompting, micro-iteration, relentless reading, and clear ceilings on what I'll outsource — keep velocity high without trading away system comprehension.

With that, you've seen how I blend **Mermaid, Miro, Cursor, DeepSeek, Claude, and Gemini** into one coherent, responsibly managed workflow. Steal whatever helps, ignore what doesn't, and — above all — **keep the developer in the driver's seat**.

---

_This guide represents a living document of AI-assisted development practices. Update as tools and techniques evolve._
