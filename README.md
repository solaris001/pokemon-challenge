# README Cove Pokemon  Challenge Laura's Solution

## Set-Up 

I am on a Macbook and therefore use Homebre for (most) installtions: 

```bash
# Brew handles Node + npm
brew install node

# Install Coveo CLI
npm install -g @coveo/cli

# Install Atomic 
npm install @coveo/atomic

# Cleanup: brew uninstall node
```

## Technical Decisions & Techical Brainstorming

### API Push vs. Web Crawler

Crawling the Pokemon database can be done via a web-crawler or through API push. For the technical challenge I will execute both approaches and compare briefly. A short summary on which approach to use when, will be added to the presentation.

Goal of this approach: Showcase platform proficiency, and API/engineering skills, and decision making (when to use what). 

How to structure this approach: 

**Primary: Web Crawler (pokemondb.net main crawl)**
- Set-up via Coveo Platform UI
- Index the full Pokemon catalog cleanly
- My "production" approach (#todo: Why?)

**Secondary: Push API**
- Manually push 5-10 Pokemon via code
- Show programmatically working with Coveo APIs
- Could be a seperate source or add to existing index (#todo: Find a good usecase here)   

Possible phrasing for presentation: "Here's the Web Crawler handling 1000+ Pokemon at scale. I also demonstrated the Push API by manually indexing a subset—useful when you need real-time updates or control over specific records."

#### How both components fit into the overall system

The two sources don't interact with each other at all during ingestion. They're independent, parallel pipelines that happen to feed the same Coveo index:
- Web Crawler → crawls pokemondb.net on its own schedule → indexes ~1028 items
- Push source → your script calls the API whenever you choose → indexes ~5–10 items

They only converge at the index and query layer. Your single Atomic search page issues one query against the whole index (or can filter to @source if you want to scope it), so items from both sources can appear together in the same result list — if their fields line up.

#### Indexing/Crawling Pokemon DB - Web Crawler

**Overview of Steps**:

- Easier, Plattform skills needed 
- Create a Web source in Coveo Platform
- Point it at pokemondb.net
- Configure rules to crawl only Pokemon pages (exclude Moves, Types, etc.)
- Extract Type and Generation as custom fields
- Run the crawler

**Documentation of Set-Up/Execution**

1. Create a project:
- Type: corporate website/blog (#todo: why? add proper explanation).
- Name: pokemon-atomic

2. Create a cloud-hosted web-source on the platform. 
- Name: Pokemon DB
- Starting URL: https://pokemondb.net
- Project: pokemon-atomic
- Security: Specific users and groups (#todo: add recruiting team)
- Documentation used: https://docs.coveo.com/en/malf0160/index-content/add-a-web-source

3. Configure cloud-hosted web crawler

Crawling Rules
- Starting URLs: #todo: why those two?
- Exclusions: Two exclusions (``/pokedex/national``, ``/pokedex/all``) are necessary, because they accidentally match the same URL pattern as real Pokémon pages (``/pokedex/pikachu``), so without excluding them they'd get indexed by mistake. The other three (``/move``, ``/type``, ``/pokebase``) are precautionary, because they wouldn't get indexed either way (#todo: why?), but excluding them explicitly matches the challenge's instructions and keeps the crawler from wasting time visiting those sections at all.
- Inclusions: "Include non-excluded pages that match at least one rule." Reasoning (from the docs): the other option, because "Include all non-excluded pages" automatically adds an all-inclusive rule in the background, which crawls and indexes the entire site (Q&A pages, move lists, item pages, etc.). That directly conflicts with the challenge requirement to include only the actual Pokémon pages and exclude everything else.

Web Scraping
- #todo: "elements to exclude" and "metadata to extract" left out for now

Advanced Settings
- no query parameters to ignore, because individual Pokédex URLs are clean, no query strings to normalize.
- no directives to override, because I do not own this page
- Crawl limit: 
  - Number of pages: 0 (Why? Following Coveos tip to index only a specific page fpr testing. Setting it to 0 ensures the crawler indexes only the specific starting URL and none of its linked pages)
  - Time: default 1000 ms (Why? Coveo only allows a crawl delay below 1000 ms when you can prove ownership of the website via a coveo-ownership-{orgid}.txt file at the site root — not possible here since it's not your site.)

Authentication
- We skip fully for now, since the Pokemon database is a public page

Content Security
- Everyone, because otherwise our API Key that we connect to a local search page would not be able so access the search results. 

#todo: this does not work yet! I need to reset to "false" and then fix it later.
ExpandBeforeFiltering
- We set ExpandBeforeFiltering on "true", 
- Where? Sources -> Pokemon DB -> More -> Edit configuration with JSON
- How?    
````
"ExpandBeforeFiltering": {
     "sensitive": false,
     "value": "true"
   }
````
- Why? Our source uses https://pokemondb.net/ and https://pokemondb.net/pokedex/national as Starting URLs because they link to every individual Pokémon page, but neither should appear in search results itself. By default, Coveo applies exclusion rules before expanding a page's links, so excluding these two URLs would have also blocked the crawler from discovering the content behind them. Setting ExpandBeforeFiltering to true reverses this order — links are discovered first, then filtering decides what gets indexed — letting us use these pages purely as crawl entry points without indexing them.
- After setting ``ExpandBeforeFiltering``on ``true``, we can add them as exclusions. But note that you need to add them matching reges rules: 
  - matches regex rule: ^https://pokemondb\.net/pokedex/national/?$
  - matches regex rule: ^https://pokemondb\.net/?$
- ...

3.1 Configure cloud-host web crawler test system

A single-page Web source (starting URL: /pokedex/moltres, levels = 0) built to iterate on Type/Generation/image extraction in seconds instead of waiting on the full ~1000-page crawl. Moltres was chosen specifically for its dual Fire/Flying typing, to confirm the Type selector correctly returns multiple values. Crawling rules and web scraping config mirror the production source exactly, so anything validated here transfers directly without translation. Advanced settings (JS rendering off, robots.txt respected) also match production for consistency. The only real difference from production is scope: one page, zero crawl depth, purely for fast local iteration before porting the finalized config to the main source.

#### Testing cloud hosted web crawler

The current Pokémon database contains 1025 Pokémon. My system extracts 1032 entries. Claude analyzed the excel table containing the 1031 entries and finds 1028 legitimate (#todo: why 1028 and not 1025) Pokémon pages. Three pages are not Pokémon pages such as:
https://pokemondb.net/
https://pokemondb.net/pokedex/national
https://pokemondb.net/pokedex/shiny

The first two can't easily be excluded from the search by the "Exclusion" configuration, because they are also the start points for the web crawler. Therefor we need to edit the ``ExpandBeforeFiltering`` section (#todo: add link to the section of this documentaation on how to set it true.)

The last one is not a startpoint for the web crawler, therefore we can easily add it as "Exclusion". 

#### Indexing/Crawling Pokemon Db - API Push

- manual, more control, API skills needed 
- Scraping site myself
- push data via API

Two story's are possible to tell in order to set the frame, why I chose to also integrate the API-Push. Because naturally the cloud hosted web crawler provides me with all necessary information: I can and actually do access all 1025 Pokémon. So if I would now add the API component, I would just get duplicates of Pokémons in my search results. That is not what I want, and furthermore it does not add any more value to the system. Therefor I need to find the scenario in which my system will actually profit from the API based web crawling. 

API based web crawling becomes useful, when I want to retrieve information that does not exist as an HTML page. In real world this would mean something like customer's proprietary database or internal API. 

#open: Secondly, I can do this (since it is more complex, I postpone if I have more time later): An alternative I'd consider if you want an even stronger story: frame the pushed content as a secured "My Team" collection (content behind a login, not public) — this lets you demonstrate Push's permission-model / security identity capability too, which most candidates won't touch. It's more work, so worth weighing against your remaining time.

##### Customer Story 

**Real-world analogy: A B2B equipment manufacturer's support portal**

The manufacturer's public website has crawlable product pages — spec sheets, manuals, troubleshooting guides — indexed by a standard Web source, same as your Pokémon pages.

But for a subset of premium/flagship equipment, there's also live **telemetry and warranty data** (firmware version, sensor diagnostics, warranty expiration, maintenance history) sitting in an internal IoT/asset-management platform. That data is exposed only via an internal REST API — there's no public webpage per machine, and it changes constantly as sensors report in. A crawler literally cannot reach it, for the same reason it can't reach PokeAPI's battle stats: **it doesn't exist as HTML anywhere.**

So Push indexes that structured API data as enrichment alongside the crawled product pages — when a technician searches for a specific machine, they get the crawled manual *and* that machine's current live status pulled from the internal system, refreshed in near real-time rather than waiting on the next crawl cycle. Same architecture, same justification, same "why not just crawl it" answer as your Legendary Pokémon stats.

Pokémon Story: Pokémon DB Search indexes pokemondb.net via a Coveo Web Crawler source, covering all ~1025 individual Pokémon pages with faceted search by Type and Generation, artwork thumbnails, and full-text search. This is complemented by a Push API source that enriches a curated set of Legendary/Mythical Pokémon with structured battle data (base stats, abilities, hidden abilities) sourced from PokeAPI — data that has no crawlable HTML equivalent, mirroring how enterprises push proprietary or API-only content (e.g. live telemetry, internal databases) alongside their publicly crawled web content. Both sources feed a single unified Atomic search interface.

##### Technical Implementation

#open: will be done once everything else is set-up

1. ...
2. ...

### Local search page cloud connection

Task: Connect your local search page to the cloud endpoint to get results when searching

Creating API Key:
- Coveo Admin Console -> Organization -> API Keys -> Add API Key
- Anonymous Search: Precisely my setup. A client-side Atomic page with no login, querying public Pokémon content. Note it's tagged "Can be public", meaning it's designed to be safely dropped into browser-side JS like your initialize() call — unlike most of the other templates.
- Name: Pokemon Challenge - Atomic Search UI
- Search Hub: PokemonChallenge
- Organization ID (given by platform, and noted to target the correct endpoint): laurapokemonchallengemcfix5o4

Initializing Index.html:
- built a minimal index.html that loads the Atomic library and theme via CDN, wraps an atomic-search-box and atomic-result-list inside an atomic-search-interface tag, and calls searchInterface.initialize() with the access token, organization ID, and organizationEndpoints (resolved via getOrganizationEndpoints() so the correct region is used automatically), followed by executeFirstSearch() to trigger an initial query on load. 
- opening the file directly via file:// blocks the ES module and fetch calls Atomic needs due to CORS restrictions, we served the page locally instead with ```npx serve .``` from the project root in the terminal. This exposes the page at http://localhost:3000, where a successful connection shows Pokémon results rendering below the search box on page load.

### Type facet filter

Task: Create a facet to filter search results by Pokemon Type.

Before this task the web-scraper JSON (platform.cloud.coveo.com -> Sources -> Pokemon DB Test -> Web Scraping -> Edit with JSON) is ```web-scraping-0.1.json```

In ```web-scraping-1.0.json``` we did following: The original config only stripped out boilerplate sections (nav, footer, ads) and extracted no metadata, so Pokémon Type had no path into the index. We added a second config object scoped only to individual Pokémon page URLs (excluding the national and shiny list pages), so the extraction never runs on non-Pokémon pages. Within that scope, a CSS selector targets the type badges in the page's "Pokédex data" table and pulls their text into a new pokemontype metadata field. Because that selector matches every badge on the page, Coveo automatically captures both values for dual-type Pokémon (e.g. Fire and Flying) rather than just one. That raw metadata still needs to be mapped to an actual multi-value facet field in the Admin Console before it can drive the Type filter on the search page.

On coveo platform ```Fields``` we added a field ```pokemontype```:
- String
- Multi-value facet

Then we co got ```Sources``` on coveo plattform:
- Choose Pokemon DB Test
- Click ```More -> View and map metadata```
- Search for ```pokemontype``` and click on it
- Click ```Add to index```
- Field: pokemontype 

Interesting observation: The Pokémon DB has the normal and the Galarian Pokémon as two panel tabs on the same page. 
So if we access the type through the metadata by this path in the JSON file ```table.vitals-table a.type-icon::text``` 
we get the types for both Pokémon back (e.g. Moltres: Fire, Flying AND Galarian Moltres: Dark, Flying). In order to fix 
that we change the path ```.sv-tabs-panel.active table.vitals-table a.type-icon::text```. That way the type is only returned
for the active Pokémon tab, the displayed Pokémon data of the normal Pokémon - not the one in the second tab. 

As last step we add the facet to ```index.html```. That way we 
### Generation facet filter

Task: Create a facet to filter search results by Pokemon Generation.

Similarly to the previous set-up. #todo: write short documentation on how and why we changed the web-scraping-1.0.json file. 

### Display Pokémon picture

Task: Display the Pokemon’s picture directly in their search result.

We are again in the Web-Scraping JSON file. There we ass the pokemonimage entry into the same metadata onject as pokemontype and pokemongeneration. 

## Backlog 
- "List of Pokémon (sprites gallery)" https://pokemondb.net/pokedex/national is still on the list
- Facet Filter "Type" returns 110 Flying Types, while officially there are 134
- Missing image on Iron Boulder: likely explains itself — recall ~10% of pages were missing this field back when we checked the metadata sample, and Iron Boulder (a "Paradox" Pokémon, which sometimes has a slightly different page layout) may be one of them. Rather than debug every edge case, Atomic actually has a documented fallback attribute for exactly this — it's even the thing that console warning has been suggesting this whole time. Let's use it instead of chasing 100% coverage.
- 