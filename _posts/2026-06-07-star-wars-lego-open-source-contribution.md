---
layout: post
title: "From Star Wars LEGO to My First Open Source Contribution"
date: 2026-06-07
categories: [neo4j, ai, open-source]
excerpt: "I wanted to build a Star Wars LEGO database. That small frustration turned into something I didn't expect: my first merged contribution to an open source repository."
---

I've always wanted to build a Star Wars LEGO database. Not for work. Just because I wanted to: every set, every minifigure, every relationship between themes and years and pieces, all of it queryable in a graph. The problem was I never had a good place to start.

Without an existing database, most tools assume you already know what you're building. I didn't. I had a pile of CSV files from Rebrickable and an idea.

That small frustration turned into something I didn't expect: my first merged contribution to an open source repository.

## A New Hope: The Force Was a Session

Last week our team ran a session with Zach Blumenfeld on his research into AIP (Agent Instruction Protocol) and how structured skills change the way AI agents perform. I'm not an engineer by trade, and working daily alongside a team like this makes me want to get my hands on things to better conceptualize new concepts.

So I did.

## These Aren't the Entities You're Looking For

What I didn't know at the time was that starting a graph project from CSV data has its own technical challenges. My plain AI agent had nothing to work from. I'd ask it to help me build a database and it would guess. Confidently, plausibly, and incorrectly.

Neo4j Agent Skills to the rescue -- these encapsulate expert knowledge on graph data modeling, cypher generation and import.

I wanted to define the schema first, before the database existed. Tell the agent: here are the nodes, here are the relationships, here are the properties, now help me build it correctly.

The result was `neo4j-dynamic-schema-guardrail`: a `SKILL.md` file and a schema snapshot that any AI coding agent picks up automatically. No server, no library, no plugin, no live connection required. The agent reads the schema file before generating any Cypher, validates every entity against it, and halts with a structured report if something doesn't exist.

I used it to load five Star Wars LEGO themes, 1,122 sets, and 1,528 minifigures into a Neo4j graph, built entirely from CSVs, with validated import scripts, before a single node was written. It worked better than I expected.

## Go See Michael

When my VP saw what I'd accidentally built, he suggested I run it by Michael Hunger, VP of Product Innovation at Neo4j, to see if the project had value for the broader Neo4j community -- specifically the neo4j-skills repository. I reached out with what I thought was a polite, low-pressure check-in.

Michael's response was immediate and direct.

> "Did you check the cypher skill we have in the skills repository? Would love to get contributions to that one."

I'd thought I was at the end of an accidental side project. That one question changed the coordinates.

Before going back to Michael, I did the work of understanding where the guardrail fit. I installed the cypher skill and ran it the way I'd been working in a session with no live database connection to see what happened.

The cypher skill is built to validate against a live database: it reads the real schema before it writes Cypher. With no connection, it has no schema to read, so it does the reasonable thing and infers one. When I asked it to find all minifigures in a set, it generated:

```
MATCH (s:Set {set_num: $setNum})-[:CONTAINS]->(m:Minifigure)
RETURN m.fig_num AS figNumber, m.name AS name, m.num_parts AS parts
```

None of those entities exist in my database. The label is `:Minifig`, not `:Minifigure`; the relationship is `:HAS_MINIFIG`, not `:CONTAINS`; the property is `id`, not `set_num`. The query is plausible, and without a schema to check against there's no way to know it's wrong. That's not the skill failing -- it's what happens to any tool when the schema isn't available.

The guardrail reads the schema from a file, so it has something to check against even with no connection:

```
Schema Validation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Node: Set          | ✅ FOUND
Node: Minifig      | ✅ FOUND
Rel: HAS_MINIFIG   | ✅ FOUND (Set -> Minifig)
Property: name     | ✅ STRING
Property: fig_num  | ✅ STRING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MATCH (s:Set {id: $setId})-[:HAS_MINIFIG]->(m:Minifig)
RETURN m.name AS minifigName, m.fig_num AS figNum, m.num_parts AS numParts
ORDER BY m.name
```

Where each tool fits:

- **graph-guard** -- real-time validation through a Bolt proxy server.
- **cypher-query-validator** -- a TypeScript library you integrate into your code.
- **neo4j-cypher-skill** -- validates against a live database connection.
- **neo4j-dynamic-schema-guardrail** -- file-based, works without a connection, define-first workflow.

I put together a full evaluation brief, a live side-by-side comparison with real test outputs, and a complete session log, and sent all three back to Michael.

That's when the real conversation started.

## Padawan Github Project Training

Over the course of one day, Michael gave me feedback that changed how I thought about the project. Not the core mechanism, but everything around it.

> "The LLM doesn't read the README. It reads SKILL.md."

> "I don't think you need the repository layout section."

> "Replace the screenshot with copyable text."

> "Does it attempt to map synonyms before halting?"

This question became a feature. If someone asks for `Minifigure` and the schema has `Minifig`, the original skill just halted. Michael asked whether it could suggest the close match first, and he was specific about the logic: if the match is unambiguous, resolve it silently and continue. Only halt if it's genuinely ambiguous.

> "Show the empty-database and import workflow clearly in 'How It Works.'"

The most valuable thing I'd built -- defining a schema from CSV before a database exists -- wasn't clearly described anywhere.

> "Add support for the schema format in the neo4j-graphrag-python package."

> "Apply AGENTS.md to condense the skill."

The principle is called "caveman compression": remove articles, politeness, conjunctions, and explanatory padding, and keep technical content untouched.

Each time Michael gave feedback, I passed it directly to Claude and pushed the updates. At one point he wrote:

> "you can just pass my feedback to Claude and let it fix it."

Which made me laugh, because that's exactly what I was doing.

## The Contribution Itself

I'd never contributed to another open source project before in a technical way. Michael's instruction was simple:

> "Make a fork of the skills repo, have Claude clone it, tell it to apply and integrate the skill into a new branch, and open a pull request. You can just paste this instruction to Claude."

So I did. It forked `neo4j-contrib/neo4j-skills`, created a branch, migrated the Python scripts and validation logic from the guardrail into the cypher skill, and opened PR #32.

## Michael's Final Edit, From the Plane

While in the air, Michael made one last change himself. He softened the strict "halt on failure" behavior into something more nuanced: only halt when the problem genuinely can't be mitigated. Otherwise, propose a resolution, ask the user if needed, and proceed.

> A guardrail that always stops is a blocker. One that tries to find a path forward first is a collaborator.

That shift was the last thing that went in before the PR merged.

## What I Learned

The code changed less than I expected. The scripts mostly stayed intact. The validation logic was largely the same. What changed was clarity: about who reads what, what belongs where, what the tool is actually for, and how it should behave when something goes wrong.

The guardrail handles the *what*: are these entities real?
The cypher skill handles the *how*: is this query well-formed?
Neither replaces the other. That's the complementary relationship.

I started out wanting to build a Star Wars LEGO database without hallucinated Cypher. I ended up with my first open source contribution, a much better understanding of how AI agent skills actually work, and a real lesson in the difference between something that works and something that's ready to share.

The original guardrail lives at [github.com/andwaller/neo4j-dynamic-schema-guardrail](https://github.com/andwaller/neo4j-dynamic-schema-guardrail), and the neo4j-cypher-skill it is now is part of [neo4j-contrib/neo4j-skills](https://github.com/neo4j-contrib/neo4j-skills).

*Originally published on the Neo4j Developer Blog, June 2026.*
