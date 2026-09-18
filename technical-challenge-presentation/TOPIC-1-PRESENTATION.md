# Pokémon Challenge

This file tells the story of the build in architecture order — the way a panel needs to hear it —
rather than the task-by-task order it was built in. For implementation detail, click-paths, and
commands, `README.md` in this repo remains the source of truth; this file is the speaking narrative
built on top of it.

**Time budget (25 min slot, incl. Q&A):**

| # | Section | Time   |
|---|---|--------|
| 1 | Solution Architecture | 4 min  |
| 2 | Indexing pokemondb.net | 3 min  |
| 3 | Metadata Extraction & Facets | 4 min  |
| 4 | Advanced Modelling | 4 min  |
| 5 | Next Steps in the System | 1 min  |
| 6 | Pokédex to Production (customer use case) | 4 min  |
| | **Buffer for panel Q&A** | ~5 min |
---

## 1. Solution Architecture 

🎤 * ~4:00 in, live demo on (https://solaris001.github.io/pokemon-challenge/)*

![Solution architecture diagram](pokemon-challenge-solution-architecture.jpg)

The system runs two independent ingestion paths into one unified Coveo index, surfaced through an
Atomic v3 front end:

- **Web Crawler (primary/production path).** Coveo's cloud-hosted web crawler is built for content that already exists 
as public web pages. pokemondb.net qualifies directly, making this the technically correct choice.
- **Push API (secondary, deferred).** Coveo's Push source is designed for content with no native connector and no public 
page to crawl. That condition doesn't hold for pokemondb.net itself, so Push isn't the right tool for the core catalog,
but it's kept as a parallel pipeline for content that *does* meet that condition (see the dashed lines in the diagram). 
The Push API makes the system open to the input side. This is imporant for thinking about Coveo as a search-layer within
an agentic system. 
- **Why they stay separate:** The two sources don't interact during ingestion, they're independent pipelines that happen 
to feed the same index. This let the crawler-based solution ship first, with Push added as a non-blocking enrichment 
layer once we want to add OpsAgents to the system.
- Default query pipeline: the same pipeline also carries the RGA and Query Suggest model associations that power the 
generative answer and type-ahead, both covered in depth in Section 4.
- **Front end & hosting**: Atomic was chosen over Headless because it ships production-ready UI components out of the box, 
letting the challenge focus on Coveo configuration depth rather than front-end build time — Headless would make more sense 
if the target app needed a fully custom React/Angular UI. The resulting index.html is version-controlled on GitHub and 
served directly via GitHub Pages, so the same file that's reviewable in the repo is what's live for the panel to try.
- **Passage Retreival**: Opens the same system to the output side. Instead of one generated paragraph (RGA), it returns 
scored, source-attributed passages that anything (an agent, a script, an LLM) can consume and act on. For real customers 
this is often the more important piece, as the closing section will show.

---

## 2. Indexing pokemondb.net

🎤 *~7:00 in*

**Scoping the crawl:**

- Two starting URLs: 
  - the site root, 
  - https://pokemondb.net/pokedex/national: the one page listing a direct link to
    every Pokémon, guaranteeing complete discovery since Coveo's crawler finds pages by following links
    from its entry points.
- Inclusion rule: 
  - "include non-excluded pages that match at least one rule," 
  - scoped to `^https://pokemondb\.net/pokedex/[a-z0-9-]+/?$` 
    - deliberately not the crawler's all-inclusive default, to satisfy "Pokémon pages only."
- Exclusions: 
  - `/pokedex/all`: would follow inlcusion regex
  - `/pokedex/shiny`: would follow inclusion regex
  - `/type`, `/pokebase`, `/move`: stability if inclusion gets loosened, not necessary
- Crawl limit: 
  - 1 levels deep (needed to reach `pokemondb.net/pokedex/<name>`)
  - default 1000 ms delay (Coveo only allows faster crawling with proven site ownership, not applicable here, since it 
  isn't my site).

**Fast-iteration workflow:** a single-page test source (starting URL scoped to one Pokémon, 0 crawl
levels) mirrors the production config exactly, so Type/Generation/image extraction can be validated
in seconds instead of waiting on a full ~1000-page crawl. Moltres was chosen deliberately for its dual
Fire/Flying typing, to confirm the Type selector correctly returns multiple values.

**Known result gap:** the crawl returns 1026 entries against an expected 1025:
    one stray page, `/pokedex/national` (a sprite gallery), is slipping past the exclusion rules. Something to fix in next iteration.

> **💡 Note:** `/pokedex/national` is both a crawl *entry point* (needed for discovery) and a page I want *excluded from 
> the index*. A plain > exclusion rule can conflict with using a page as a starting point. `ExpandBeforeFiltering` (a
> JSON-config override) lets the crawler follow links out of the page without indexing the page itself. 
> Adding ``ExpandBeforeFiltering`` will turn the crawl into a multi-hour rebuild > by forcing full-link checks on every 
> excluded page sitewide. Therefor this implementation step has been postponed for now.

---

## 3. Metadata Extraction & Facets

🎤 *~10:00 in, facet demo checkpoint here.*

**The core problem:** the original web-scraping config only stripped boilerplate (nav, footer, ads)
and extracted no metadata: The type and generation had no path into the index. The fix: more config object scoped
only to individual Pokémon page URLs (excluding the national/shiny list pages), so extraction never
runs on non-Pokémon pages. Inside that single scoped block, three selectors cover everything the Essential checklist 
asks for:
- pokemontype (CSS): ``.sv-tabs-panel.active table.vitals-table a.type-icon::text``, targeting the type badges in the "Pokédex data" table. 
- pokemongeneration (XPath): ``//p[contains(., "introduced in")]//abbr[starts-with(@title, "Pokémon ")]/text()``, reading the generation label out of the introductory paragraph's abbreviation tag (its title names the originating games).
- pokemonimage (XPath): ``//div[contains(@class,"sv-tabs-panel") and contains(@class,"active")]//img[contains(@src,"/artwork/")]/@src`` pulling the official artwork image's src rather than a sprite thumbnail.

**The debugging story (your strongest one):** pokemondb.net renders the normal and Galarianor other forms of
some Pokémon as two tabs on the *same page*. The naive selector,
`table.vitals-table a.type-icon::text`, pulled types from *both* tabs — e.g. Moltres returning Fire,
Flying, and Dark (Galarian Moltres' type bleeding in). Scoping it to
`.sv-tabs-panel.active table.vitals-table a.type-icon::text` restricts extraction to only the
active/displayed tab, fixing it.

**Facet semantics decision:** Type uses Coveo's default `resultsMustMatch: atLeastOneValue`
selecting Dark and Ghost returns Pokémon matching *either*, not both. This was deliberate, not an
oversight: it preserves the checkbox-filter mental model users bring from virtually every
e-commerce/enterprise search UI, where more selections broaden results. An "all values must match"
mode would support exact dual-type lookups (a real, common query in this domain) but was rejected as
the *default*, since it inverts a standard UI convention and caps out at two selections before
guaranteeing zero results.

---

## 4. Advanced Modelling

🎤 *~15:00 in PM, demo checkpoint here: RGA-generated answer, then a Query Suggest type-ahead.*

**RGA (Relevance Generative Answering):**
- RGA ist Coveo's Retrieval Augmented Generation systems. It is grounding a generative LLM answer with own content 
(from the unified index). Coveo-specific is, that it has two stages: 
  1. Query runs through the normal Coveo search engine   and query pipeline and gets the top results. Only those top results are then handed to the RGA model. 
  2. The RGA model   uses embeddings to find the most relevant chunks of text within them. 
  
        (Excursion: THe LLM never sees the whole index, only pipeline filtered, already permissioned content relevant to that query. Step 1 is also security relevant, because Coveo
  controls by that exactly what conent the LLM is seeing. This also helps agains hallucinations, because the generated answer
  stays confined to intexed content and it's existing permission structure.)
- Learns from the **Pokemon DB** source; no filter needed since the source only contains indexed Pokémon
  pages.
- Associated to the query pipeline under `Query is not empty: RGA generates its answer from the
  query text, so on an empty query there's nothing to embed or retrieve against.
- `Items to consider: 100`: caps how many top-matching results (from the initial normal search) get
  passed into the second stage, where passages are actually pulled. Generous headroom for a
  ~1,028-item index; would only shrink with a finely-tuned pipeline forcing the model to ignore
  lower-ranked results.
- `Chunk relevancy threshold: Medium`: how picky the model is about which text chunks count as
  relevant enough to use. Too high and vague queries generate nothing; too low and marginally-related
  text gets used. Medium is the balanced default.
- `Rich text formatting: On`: Atomic picks this up automatically, no extra config needed; it's just
  a nicer-rendered answer for the same content.
- `Thesaurus rules: Off`: only matters if the pipeline has synonym rules bridging vocabulary gaps
  (e.g. "flying-type" → "Flying"). Since Pokémon names/types/generations already match how users
  search, there's no gap to bridge.
- On the front end, this is just one line: I added the `<atomic-generated-answer>` component inside my atomic-search-interface.
Atomic automatically wires it to whatever RGA model is associated with that interface's query pipeline, so there's no extra 
JavaScript or manual API wiring needed — the component handles fetching and rendering the generated answer on its own, 
the same way atomic-result-list does for normal results.


**Query Suggest:**
- Learns from usage analytics (real queries and clicks, localhost and github app) a brand-new model has no history, so it'd
  show an empty dropdown on day one.
- Preloaded via a Default Queries CSV (AI created list of expected search terms) so type-ahead
  works immediately in the demo instead of waiting for organic usage data.
- Associated with the pipeline under *no condition*: unlike RGA, suggestions should fire on any
  partial input, so there's no equivalent restriction needed.
- Uploaded via a separate, more privileged API key (Machine Learning: Models: Edit, kept out of any
  client-facing file), used once from the terminal, distinct from the public Anonymous Search key in
  `index.html`.
- On the front end, this is just one line: I added the `<atomic-search-box-query-suggestions>`

---

## 5. Next Steps in the System

🎤 *~20:00 in PM*

Self-flagged, roughly in order of visibility to a live demo:

- **Type facet undercount:** the Flying-type facet returns 109 results against an official count of
  134.
- **`/pokedex/national` still indexed** despite exclusion rules — tied to the `ExpandBeforeFiltering`
  question flagged in Section 2.
- **Facet result ordering:** an enhancement idea to surface perfect (both-type) matches first when
  two Type values are selected.
- Tatsugiri image not depicted
- "List of Pokémon (sprites gallery)" page still there
- Increase UX

---

## 6. Pokédex to Production (customer use case) & the Passage Retrieval API

🎤 *9.20 PM*

For the final part of the presentation I want to combine two sub-tasks of the presentation: I want to build a system analogy
from the Pokémon Challenge solution to a customer that I worked with recently, AND I want to show how we would extend that
system productively using the Passage Retreival API. 

**Passage Retrieval** takes a natural-language query and returns a ranked list of short text passages pulled directly from 
the indexed content. It does not return whole documents or written answer. It works in two steps: it first finds the most 
relevant items using semantic (meaning-based) search, then extracts the specific passages within those items that actually 
address the query, scoring each one and linking it back to its source. Because it hands back scored, sourced evidence 
rather than a finished answer, it's built to be consumed by something else: an app, an agent, or an LLM, which decides 
what to do with those passages.

**The customer:** 
- A workforce-management software provider building an AI Ops Agent (with Agentforce/AgentCore/Joule for example) for 
their technical hotline support. 
- Three-month pilot. 
- Knowledge base: manuals stored in S3, historical Jira tickets with the LLM-generated FAQ/solution hint stored
  directly on the ticket, and Confluence planned as a later addition.
- The agent reads incoming tickets via a tag-trigger, fetches context from KB1 (tickets/FAQ) or KB2 (manuals, as
  fallback), and escalates to a human when confidence is low.

**There data mapped onto the "Pokémon" Solution**
- **Manuals, stored in S3** → same principle as the Web Crawler, different connector: content that already lives somewhere 
    Coveo can reach directly. Coveo has a native Amazon S3 source for exactly this. Point it at the bucket, same "index 
    it where it lives" logic as crawling pokemondb.net.
- **Several thousand historical Jira tickets, with the LLM-generated FAQ/solution hint stored directly as the answer on 
the ticket** → this is a native-connector case, not Push. Coveo's Jira connector reaches whatever's on the ticket, including 
that generated-answer field, the same way it reaches the description or comments. The real work here mirrors something 
from the Pokémon challenge: Mapping that generated-answer field into a proper Coveo metadata field once indexed, the same 
move as pulling `pokemontype` out of pokemondb.net's raw HTML.
- **Confluence (planned, later)** → a second native connector, added without touching the first two sources. Same property 
  the dual-source setup from the solution architecture diagram already demonstrates: sources can be independent, and
  the unified index doesn't care how many feed it.

**Where Passage Retrieval does the actual work:**
- Their "fetch context, then answer" step maps to Passage Retrieval, not RGA: a normal search finds the top candidate
  tickets/manuals, Passage Retrieval extracts and scores the specific passages within them, and hands those (not a
  finished answer) back to the agent.
- Their "escalate to a human on low confidence" requirement is exactly what those scores are for: the agent's own
  Topic guidelines (in Agentforce, for example) define what score counts as confident enough to answer directly
  versus hand off, no new capability to build on the Coveo side, just a score already being returned.
- This is why Passage Retrieval, not RGA, sits in the solution diagram feeding the "AI Agent" box: an agent
  platform owns the generation and the escalation decision itself, so it needs raw, scored, sourced material to
  reason over, not an answer Coveo already wrote. Coveo for Agentforce ships a Passage Retrieval action for exactly
  this reason.

**Push Source — closing the loop:**
Coveo's role in this design stops at retrieval and a confidence score; the escalation decision itself belongs to the
orchestration layer (Agentforce, AgentCore, Joule, or similar). Once that layer is running, each execution produces a 
new fact: The query, the confidence score, the passage used, whether it escalated, the eventual resolution (that has no 
home in Jira, S3, or Confluence). That's exactly the shape Push is for: no crawler or connector applies, because the data 
doesn't exist until the orchestration logic produces it. Once pushed and indexed, those records become queryable by 
confidence score or outcome, giving the pilot a way to actually measure its own automation rate instead of assuming one.

**The value proposition, tied to their actual problem:** This setup grounds the Ops Agent in the same knowledge a support 
engineer would use: Manuals and previously solved tickets. The Ops Agent answers from current content instead of guessing, 
and hands off to a human whenever its confidence says it shouldn't. That means faster and better resolution on the tickets 
it can handle, and no wrong answers reaching a customer on the ones it can't. And because every run leaves a scored, 
queryable record behind, the automation rate becomes something the customer can actually measure and improve over time. 

**Why Passage Retrieval, and not RGA**:
- This customer asked for decisions: Routing between KBs, escalating on low confidence, not only generated text. 
- RGA always writes one finished answer; it has no mechanism to route or escalate. 
- RGA only draws on Coveo's own index. A real reply needs live ticket context (product version, customer tier) blended 
in too, which an agent working from raw passages can do, RGA can't.
- Since the customer is working on an "Ops Agent" implies eventually taking actions, not just answering. RGA's ceiling is text; 
an agent architecture can grow into that from day one. 
- The real distinction isn't retrieval quality, it's who owns the decision: and this customer asked for an agent, not a search box.

**Summary**: This system was already agnostic on the input side,any content source can feed the unified index, whether 
it's reachable directly through a connector or has to be pushed in because it doesn't exist as a document anywhere. 
Passage Retrieval makes the same true on the output side: it doesn't generate an answer or assume who's asking, it just 
returns scored, sourced passages to whatever calls it — an agent, a script, anything. That's the productive extension: 
Coveo stays in the middle doing retrieval and relevance, while what feeds it and what consumes it can both change without 
touching that core.
---

*Implementation detail, commands, and click-paths live in `README.md`, this file is the narrative built on top of it.*
