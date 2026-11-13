# MRE Setup Guides

Free, copy-paste setup guides for connecting the tools we use with **Claude Code**.

Each guide stands alone — there's nothing to sign up for here. Open the one you need, follow
the steps, done. Built for agents and teammates getting their own tooling running.

## Guides

- **[Connect Lovable to Claude Code](connect-lovable-to-claude-code.md)** — wire Claude Code to
  Lovable's official MCP server so it can read your project, query your database (including the
  schema/RLS changes the Cloud UI blocks), and deploy — all via a one-time browser login. No API
  keys.
- **[Connect Your Lofty to Claude Code](connect-lofty-to-claude-code.md)** — build a small MCP
  server so Claude can read your own Lofty CRM in plain English (this week's leads, a lead's notes,
  a listing search). Uses the API key already in your Lofty settings, read-only first, security
  rules up front. A [developer-key variant](connect-lofty-to-claude-code-developer-key.md) is linked
  inside for accounts that need it.

## Ready-made files

- **[`.mcp.json`](.mcp.json)** — drop this into the root of your own project and Claude Code will
  offer to enable the Lovable MCP server (you still do the one-time `/mcp → Authenticate` login).

_More coming (Linear, Semrush, and other connectors)._

> Handling an API key that can read your CRM? Treat it like your online-banking login: it lives in a
> local `.env` that is never committed, start read-only, and reset it in your tool if it ever leaks.
> Each guide says this where it matters.

---

Free to use and share.

---

Maintained by Katie Van Nes of Most Referred Expert. The Lofty setups get taught live and free
every week at [LoftyMasterClass.com](https://LoftyMasterClass.com), and the recordings live in the
Most Referred Expert Community.
