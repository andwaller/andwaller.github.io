---
layout: post
title: "The Skill in Action: From CSV to Star Wars Graph with Claude Code and Zed"
date: 2026-06-26
categories: [neo4j, ai, open-source]
excerpt: "From a blank Zed window to a queryable Star Wars graph -- no Cypher knowledge required."
---

<style>
figure { margin: 2em 0; text-align: center; }
figure img { max-width: 100%; width: 100%; display: block; margin: 0 auto; border: 1px solid #ddd; border-radius: 4px; }
figcaption { margin-top: 0.6em; font-size: 0.85em; color: #666; font-style: italic; text-align: center; }
.instruction { display: flex; align-items: flex-start; gap: 14px; background: #f0f7ff; border-left: 4px solid #0369a1; padding: 13px 16px; margin: 1.4em 0; border-radius: 0 6px 6px 0; font-size: 1em; line-height: 1.5; }
.step-label { display: inline-flex; align-items: center; justify-content: center; min-width: 26px; height: 26px; background: #0369a1; color: white; border-radius: 50%; font-size: 0.8em; font-weight: 700; flex-shrink: 0; margin-top: 2px; }
</style>

If you read [Part 1](/2026/06/07/star-wars-lego-open-source-contribution.html), you know how this started: I wanted to build a Star Wars LEGO database, my AI agent kept hallucinating schema that didn't exist, and fixing that problem turned into my first open source contribution to `neo4j-contrib/neo4j-skills`.

That post told the story. This one shows the work.

Here's why it's worth following along: if you've ever asked an AI agent to help you build a database and ended up with queries that run but return nothing -- because the labels are wrong, the relationship types are invented, the properties don't match -- this is the workflow that solves that. The agent reads a schema file before writing a single line of Cypher. You define it once from your own data. Everything after is validated against it.

I'm not an engineer. I did this entirely in plain English, with Claude Code running alongside the project in Zed. No Cypher knowledge required -- I still don't have any.

Now let's walk through it.

## What the contribution actually did -- in plain English

The Neo4j cypher skill already knew how to write Cypher. What it didn't have was a way to work when there was no database to learn from.

The skill needs a live connection to read the schema -- what nodes exist, what relationships exist, what properties they have. Without that, it guesses. And as I showed in Part 1, it guesses wrong. It wrote `:Minifigure` when the label was `:Minifig`. It wrote `:CONTAINS` when the relationship was `:HAS_MINIFIG`. Plausible. Incorrect. No way to know the difference without a schema.

What PR #32 added was simple: **read the schema from a file first.** Before the database exists. Before you've written a single import script. Before you know anything about Cypher yourself.

That one change means a complete beginner can now:

- Describe their data in plain English and get a schema file written from their CSVs automatically
- Have every Cypher query validated against it -- labels, relationships, properties, directions
- Import their data correctly the first time
- Query it in plain English without knowing the query language

You don't need to know what a node is. You don't need to know what a relationship type looks like. You don't need an existing database to copy from. The skill holds all of that knowledge. You just have to show up with your data.

## What you need

Everything in this walkthrough is free. I know because I checked before I started -- I wasn't building something that required a subscription or a credit card to follow along.

**Zed** -- the code editor we're using. Download at [zed.dev](https://zed.dev). Available on macOS, Windows, and Linux.

**Node.js / npm** -- required to run the `npx skills add` command. Download at [nodejs.org](https://nodejs.org) (LTS version). Check with `node --version` in your terminal.

**Python** -- required to run the import scripts. Download at [python.org](https://python.org) (version 3.8 or later). Check with `python --version` in your terminal.

**Claude Code** -- the AI agent that does the heavy lifting. Install instructions at [claude.ai/code](https://claude.ai/code). Requires a Claude Pro or API account.

**Neo4j Aura Free** -- your free cloud graph database. Sign up at [console.neo4j.io](https://console.neo4j.io). No credit card required.

**The Rebrickable CSVs** -- free to download at [rebrickable.com/downloads](https://rebrickable.com/downloads).

That's the whole list. No local database, no server, no infrastructure to manage.

## Step 1: Install the skill

<div class="instruction"><span class="step-label">a</span> Open Zed and hit <code>Ctrl+`</code> to open the built-in terminal.</div>

<div class="instruction"><span class="step-label">b</span> Run this command:</div>

```
npx skills add neo4j-contrib/neo4j-skills/neo4j-cypher-skill
```

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss01-install-command.png" alt="Zed terminal showing the npx skills add command with install prompt">
<figcaption>Installing the neo4j-cypher-skill directly from Zed's built-in terminal -- no context switching required.</figcaption>
</figure>

<div class="instruction"><span class="step-label">c</span> The CLI asks which agent to install to. Select <strong>Zed</strong>.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss02-agent-selector.png" alt="The skills CLI agent selector showing all supported coding agents">
<figcaption>The list includes Amp, Cline, Cursor, Codex, Gemini CLI, Zed, and others. Select Zed -- the skill will wire itself in automatically.</figcaption>
</figure>

<div class="instruction"><span class="step-label">d</span> When asked for installation scope, choose <strong>Project</strong>. The skill travels with your repository -- anyone who clones it gets the same agent behavior.</div>

The CLI runs a security assessment before installing. When it completes, the skill is ready.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss04-install-complete.png" alt="Installation complete screen showing security assessment and file locations">
<figcaption>Safe, 0 alerts, Low Risk. Files land in <code>~\.agents\skills\neo4j-cypher-skill</code> and Zed picks them up automatically.</figcaption>
</figure>

## Step 2: Get the data

The dataset comes from [Rebrickable](https://rebrickable.com/downloads/) -- a community-maintained LEGO database with every official set, minifigure, part, and color as freely downloadable CSVs. No account required, updated daily.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss05-rebrickable-downloads.png" alt="Rebrickable LEGO Catalog Database Downloads page showing schema diagram and file list">
<figcaption>The schema diagram on the left shows exactly how the data is structured before you write a single line of Cypher.</figcaption>
</figure>

That schema diagram is worth a moment. It shows the relational shape of the data: sets belong to themes, inventories link sets to their minifigs. This is exactly what we're going to turn into a graph.

<div class="instruction"><span class="step-label">a</span> Go to <a href="https://rebrickable.com/downloads">rebrickable.com/downloads</a> and download these five files -- click <strong>zip</strong> next to each:
<ul style="margin-top:8px;margin-bottom:0">
<li><code>themes.csv</code></li>
<li><code>sets.csv</code></li>
<li><code>minifigs.csv</code></li>
<li><code>inventories.csv</code></li>
<li><code>inventory_minifigs.csv</code></li>
</ul>
</div>

<div class="instruction"><span class="step-label">b</span> Create a folder called <code>lego-starwars-csvs</code>. Move the five zip files in. Select all with <strong>Ctrl+A</strong>, right-click, and choose <strong>Extract All</strong> into the same folder.</div>

<div class="instruction"><span class="step-label">c</span> Open the folder in Zed with <strong>Ctrl+K, Ctrl+O</strong> and select <code>lego-starwars-csvs</code>.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss09-zed-project-open.png" alt="Zed project sidebar showing five CSV files with database icons">
<figcaption>Project open in Zed. Five CSVs in the sidebar -- Zed recognises them as structured data.</figcaption>
</figure>

## Step 3: Define the schema -- before the database exists

**And to be clear about what's happening here: there is no Neo4j database running.** No Aura instance, no local server, no connection string. Just CSV files and the skill.

That's the gap the PR fills. Without a schema file, the cypher skill infers the schema -- which is where hallucinated labels and wrong relationship types come from. With a schema file, every query is validated before it runs.

<div class="instruction"><span class="step-label">a</span> Open Zed's terminal with <code>Ctrl+`</code>, type <code>claude</code>, and hit Enter.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss11-claude-code-welcome.png" alt="Claude Code welcome screen showing version, model, and working directory">
<figcaption>Claude Code v2.1.181, Sonnet 4.6, working directory lego-starwars-csvs. Ready.</figcaption>
</figure>

<div class="instruction"><span class="step-label">b</span> Type this prompt and hit Enter:</div>

```
Use define_schema.py from the neo4j-cypher-skill to define a schema 
for a Star Wars LEGO graph database using the CSV files in this project. 
I want nodes for Theme, Set, and Minifig with relationships between them.
```

Claude Code reads the CSVs, builds the graph model, and asks permission to write `lego-starwars-schema.json`.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss13-schema-output-permission.png" alt="Claude Code showing schema output with relationships and permission prompt">
<figcaption>Claude Code read the CSVs, built the graph model, and is asking permission to write the schema file.</figcaption>
</figure>

<div class="instruction"><span class="step-label">c</span> Hit <strong>Yes</strong> to approve writing the schema file.</div>

It writes the file and explains what it modelled and why -- including the key insight that `inventories` and `inventory_minifigs` are join tables that become the `INCLUDES_MINIFIG` relationship with quantity as its property, not separate nodes.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss14-schema-complete-graph-model.png" alt="Schema complete screen showing graph model table with nodes, properties, source CSV and join paths">
<figcaption>The graph model table. Nodes, key properties, source CSV, and join paths. 123 lines written to import.cypher.</figcaption>
</figure>

**No database. No connection string. No server running.**

The schema file now exists. Every Cypher query Claude Code generates from this point forward is validated against it -- not guessed.

## Step 4: The Empire Strikes Back -- the first roadblock

Claude Code also wrote `import.cypher` alongside the schema. When you prompt it to run that against a live instance, it stops before executing a single line.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss16-file-path-issue.png" alt="Claude Code catching the file path issue before running anything">
<figcaption>"file:/// paths in LOAD CSV don't work on AuraDB." Caught before running anything.</figcaption>
</figure>

`LOAD CSV` with `file:///` paths works against a local Neo4j instance because the database can read your disk. AuraDB is cloud-hosted -- it has no access to your local filesystem. Rather than fail, Claude Code reasoned through it and pivoted entirely:

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss17-python-pivot.png" alt="Claude Code writing import_aura.py using the Python neo4j driver">
<figcaption>Python + the Neo4j driver is already installed. Claude Code rewrote the import as 187 lines of batched UNWIND + MERGE -- reads CSVs locally, pushes data through the driver. No file path problem.</figcaption>
</figure>

This is the kind of thing that costs an hour of Stack Overflow searching if you hit it alone. Claude Code caught it, explained it, and fixed it without being asked.

## Step 5: Set up a free Neo4j Aura instance

<div class="instruction"><span class="step-label">a</span> Go to <a href="https://console.neo4j.io">console.neo4j.io</a> and sign up with Google or GitHub. No credit card required.</div>

The onboarding builds a live graph of your answers as you fill in the form -- you're already thinking in nodes and relationships before you've written a single line of Cypher.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss19-aura-onboarding-graph.png" alt="Aura onboarding showing a graph being built from form answers">
<figcaption>Aura builds a graph of your onboarding answers in real time. A nice preview of what's coming.</figcaption>
</figure>

<div class="instruction"><span class="step-label">b</span> Go to <strong>Instances &rarr; Create instance</strong> and select <strong>Free</strong>.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss22-aura-free-tier.png" alt="Aura instance tier selection showing Free tier selected at $0/hour">
<figcaption>AuraDB Free: up to 200k nodes and 400k relationships. $0/hour. Note: auto-deleted after 30 days of inactivity -- keep it active or export your data.</figcaption>
</figure>

<div class="instruction"><span class="step-label">c</span> Name your instance <code>StarWars_Lego_DB</code> and click <strong>Create</strong>.</div>

<div class="instruction"><span class="step-label">d</span> &#9888;&#65039; <strong>Critical:</strong> Aura shows your password exactly once. Click <strong>Download and continue</strong> before closing this screen -- the downloaded file has your URI, username, and password.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss24-credentials-warning.png" alt="Credentials screen with password warning saying it won't be available after this point">
<figcaption>Download the credentials file before you close this. You will not see the password again.</figcaption>
</figure>

Within about 60 seconds your instance is running.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss25-instance-running.png" alt="Aura instances page showing StarWars_Lego_DB running with 0 nodes and 0 relationships">
<figcaption>StarWars_Lego_DB -- RUNNING. 0 nodes, 0 relationships. About to change.</figcaption>
</figure>

## Step 6: Run the import

<div class="instruction"><span class="step-label">a</span> Back in Claude Code, tell it to run the import against your instance. Select <strong>Neo4j AuraDB</strong>, then <strong>"I'll provide credentials now"</strong> and paste in your URI, username, and password from the downloaded credentials file.</div>

Claude Code runs `import_aura.py` and reports back.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss28-import-complete.png" alt="Import complete table showing all node and relationship counts">
<figcaption>Import complete. 494 Theme nodes, 27,195 Set nodes, 16,989 Minifig nodes. 2 minutes 11 seconds.</figcaption>
</figure>

<div class="instruction"><span class="step-label">b</span> Verify in the Aura Query editor -- go to your instance, click <strong>Connect &rarr; Query</strong>, and run the verification query Claude Code provides.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss29-aura-nodes-confirmed.png" alt="Aura Query editor showing database information with Minifig, Set, Theme labels">
<figcaption>44,678 nodes. 50,937 relationships. Every label and relationship type exactly as defined in the schema file.</figcaption>
</figure>

## Step 7: These aren't the Star Wars themes you're looking for

The Rebrickable CSVs contain every LEGO set ever made. We only want Star Wars.

<div class="instruction"><span class="step-label">a</span> Ask Claude Code in plain English:</div>

```
Find all themes in the database that are related to Star Wars
```

It wrote and ran a graph traversal query using the `HAS_PARENT` relationship hierarchy. Initial result: 2 themes. I pushed back -- I knew there were more than two.

Claude Code investigated with a diagnostic script and found the bug: the original query's `NOT EXISTS` guard was too strict. There are actually 5 nodes named "Star Wars" in the data -- the root theme plus sub-themes under Technic, Advent, and Mindstorms that were silently excluded.

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss32-five-themes-found.png" alt="Five Star Wars themes table showing all ids and parent themes">
<figcaption>All 5 Star Wars themes. The "hidden" three were Star Wars licensing applied to Technic, Advent calendars, and Mindstorms -- children of those parent themes, not of the Star Wars root.</figcaption>
</figure>

I pushed back using nothing but domain knowledge. I didn't write a query. I didn't debug the code. I just knew there were more Star Wars themes and said so. The agent investigated, found its own mistake, and corrected it.

*(I'd built this database once before and found 4 themes. Now there are 5 -- Rebrickable updates daily. The data reflects the real world.)*

<div class="instruction"><span class="step-label">b</span> Ask Claude Code to clear the database and re-import with only Star Wars data.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss34-filter-chain-code.png" alt="Filter chain code showing Python filtering each CSV before touching the database">
<figcaption>The filter chain: themes &rarr; sets &rarr; inventories &rarr; inventory_minifigs &rarr; minifigs. Every CSV filtered in Python before a single node touches the database. Nothing non-Star Wars can leak in.</figcaption>
</figure>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss35-starwars-only-import.png" alt="Star Wars only import complete showing 5 themes, 1124 sets, 1529 minifigs">
<figcaption>Pure Star Wars. 5 themes, 1,124 sets, 1,529 minifigs, 2,276 INCLUDES_MINIFIG relationships. Down from 27,195 sets and 16,989 minifigs.</figcaption>
</figure>

## Step 8: These are the droids you're looking for

<div class="instruction"><span class="step-label">a</span> Ask Claude Code your question in plain English:</div>

```
Since I don't know Cypher and I want to use the power of a Graph Database 
where relationships are first class citizens -- tell me which Star Wars set 
has the most minifigures?
```

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss36-death-star-answer.png" alt="Claude Code answer showing Death Star result with Cypher and plain English explanation">
<figcaption>Winner: the 2008 Death Star (10188-1) with 48 minifigures across 23 distinct characters. The graph walked relationships directly -- no table joins, no intermediate scans. The 2025 Death Star flips it: 39 distinct characters but only 40 total, meaning almost every minifig in that box is a different person.</figcaption>
</figure>

<div class="instruction"><span class="step-label">b</span> Ask Claude Code to write a visualization query. Paste the result into the Aura Query editor and switch to <strong>Graph</strong> view.</div>

<figure>
<img src="/assets/posts/star-wars-lego-walkthrough/ss37-death-star-graph.png" alt="Neo4j graph visualization showing Death Star as hub node with minifig nodes radiating out">
<figcaption>The 2008 Death Star as a hub node with every minifig radiating out via INCLUDES_MINIFIG edges. Han Solo appears twice -- two distinct minifig variants with different head molds. The graph captures that automatically.</figcaption>
</figure>

## What this actually gives you

From five CSVs to a queryable, visualizable Star Wars graph -- no Cypher knowledge required, no database needed to define the schema, and every query validated against a file that existed before the first node was written.

The workflow in one line: **define &rarr; import &rarr; query**, with the schema as ground truth the whole way through.

I started this wanting to answer one question: which Star Wars set has the most minifigures? I ended up contributing to an open source skill that makes the whole workflow possible for anyone starting from scratch.

Install it with one command:

```
npx skills add neo4j-contrib/neo4j-skills/neo4j-cypher-skill
```

Drop a `<db-name>-schema.json` in your project root. Point Claude Code at it. The rest follows.

*The 2008 Death Star has been sitting one-third built on my shelf since 2008. Maybe now that it's fully assembled in a graph database, I'll finally finish the real one.*

*The original guardrail project: [github.com/andwaller/neo4j-dynamic-schema-guardrail](https://github.com/andwaller/neo4j-dynamic-schema-guardrail)*
*The cypher skill it became: [neo4j-contrib/neo4j-skills](https://github.com/neo4j-contrib/neo4j-skills)*
