---
mode: how-to
audience: operator
topic: plugins
---

# smart-lookup

Recommended, paired with [smart-connections](smart-connections.md). Both are by the same author (brianpetro) and **share one embeddings index**, approaching semantic search from opposite directions: Smart Connections surfaces notes *similar to the one you're looking at*; Smart Lookup is **question-first** — type a natural-language question, get the most relevant vault notes back.

Like Smart Connections, it's a **parallel peer to Memoria, not a wired component**: agent retrieval goes through Hermes tools (`qmd` and the search skills in [workflows/README.md](../../workflows/README.md)), not this plugin. Use it for human navigation ("what do I have on receptivity detection?"); don't feed it into agent workflows.

Load-bearing settings: none specific to Memoria. Because it shares Smart Connections' index, configure the embeddings once — point them at the cheap model tier, not Claude (see [smart-connections](smart-connections.md)). Installing Smart Lookup without Smart Connections is unusual; the pairing is the point.

> **Verify currency before relying on this.** The details here are carried forward from Memoria's earlier design notes; confirm Smart Lookup is still maintained and that its index-sharing with Smart Connections holds in the installed versions before treating it as a standard install.
