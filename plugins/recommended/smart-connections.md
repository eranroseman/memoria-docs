---
mode: how-to
audience: operator
topic: plugins
---

# smart-connections + markdb-connect

Optional. No load-bearing settings; default configurations are fine. If using `smart-connections` for vector search, ensure the embeddings model matches Memoria's [model routing config](../../architecture/capability-stack.md#model-routing-synthesis-on-claude-cheap-tasks-elsewhere) — embedding calls should go to the cheap model tier, not Claude.

`smart-connections` is best treated as a **parallel peer to Memoria, not a wired component**. Use it for synchronous in-Obsidian semantic search (mid-draft "what notes do I have on X?"); do **not** try to feed its embeddings into Memoria's Curator or Mapper skills. The integration paths — community MCP server or reading `.smart-env/*.ajson` directly — are fragile; the storage format is undocumented and changes between releases. Memoria's retrieval substrate (`qmd` and the search skills in [workflows/README.md](../../workflows/README.md)) is purpose-built for agent use; Smart Connections is purpose-built for human use. Use both, separately. Your attention is the integration layer.

If a cross-system bridge ever becomes genuinely necessary, the only practical option is the community-maintained `msdanyg/smart-connections-mcp` server, which exposes Smart Connections' embeddings over MCP by reading the same `.smart-env/*.ajson` files. Caveats: single contributor, small footprint, format-dependent — it ships as a community optimization, not as load-bearing infrastructure. Adopt only if Smart Connections is already a daily driver *and* a concrete agent use case appears that can't be served by Memoria's own retrieval. Otherwise the cost (forking and maintaining a bridge against an undocumented format) outweighs the saved embedding compute.

`markdb-connect` links Zotero items to vault notes — useful for the [ingest workflow](../../workflows/upstream/ingest.md). Default configuration is fine; no load-bearing settings.

<!-- memoria-nav -->

---

[← Previous: obsidian-Linter — the footgun](obsidian-linter.md)

[Next: supercharged-links + obsidian-style-settings →](supercharged-links.md)
