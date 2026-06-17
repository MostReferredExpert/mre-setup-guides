# Connect Lovable to Claude Code

This wires Claude Code to **Lovable's official hosted MCP server** so Claude can read your
Lovable project, run database queries (SELECTs *and* schema/RLS changes the Cloud UI won't let
you do), and deploy — all from your terminal or editor. It uses a one-time **browser login
(OAuth)**, so there are no API keys to copy or store.

> It connects to *your own* Lovable account. After you log in, you'll only see the Lovable
> projects your account already has access to.

## Before you start

- **Claude Code** installed and reasonably up to date (needs HTTP-transport + OAuth support).
  Check with `claude --version`; update with `claude update` if it's old.
- A **Lovable account** that owns (or is a member of) the project you want Claude to work on.

---

## Option A — one command (recommended)

Run this in a terminal:

```bash
claude mcp add --transport http lovable https://mcp.lovable.dev -s user
```

- `lovable` is just the name it'll show up as.
- `-s user` makes it available in **every** project on your machine. Drop the `-s user` if you
  only want it in the current folder.

Then **authenticate** (one time): open Claude Code and run

```
/mcp
```

Select **lovable → Authenticate**. A browser opens → log into Lovable → approve → it redirects
back and connects.

**Verify:**

```bash
claude mcp list
```

You should see:

```
lovable: https://mcp.lovable.dev (HTTP) - ✔ Connected
```

That's it. ✅

---

## Option B — drop-in project file (good for sharing with a team)

Instead of the command, you can commit a file named **`.mcp.json`** to the root of your project.
When you open Claude Code in that folder it detects the file and offers to enable the server;
you then authenticate the same way (`/mcp` → lovable → Authenticate).

Create `.mcp.json` with exactly this:

```json
{
  "mcpServers": {
    "lovable": {
      "type": "http",
      "url": "https://mcp.lovable.dev"
    }
  }
}
```

Anyone who clones the project gets the server definition; each person still does their own
one-time OAuth login, so no secrets are shared.

---

## What it unlocks once connected

- **Inspect** — read your project, files, diffs, and message history.
- **Database (`query_database`)** — the big one: query your Supabase / Lovable Cloud database
  directly, including migrations and RLS fixes the closed Cloud UI blocks. Reads and SQL like
  this don't cost credits.
- **Deploy** — publish the project to production.
- ⚠️ **Credit-costing tools** — `send_message` (the AI editor) and `create_project` **spend
  Lovable credits**. Reads and database queries don't; just know which is which.

## Two things worth knowing

- The login is **interactive**, so the connection lives in an interactive Claude Code session —
  it may not be present in fully headless/automated runs.
- If you'll have Claude change **code**, the cleanest workflow is: let Claude edit locally, push
  to GitHub, and let Lovable auto-sync from the repo — rather than driving edits through
  `send_message`. That keeps your Git repo as the single source of truth. Use `query_database`
  for **database** changes (safe, no credits).

## Troubleshooting

- **Stuck at login / "not connected":** run `/mcp` again and re-trigger **Authenticate** — it
  re-opens the browser flow. This is the most common stumble.
- **Command not recognized:** your Claude Code is likely outdated — run `claude update` and retry.
- **Want to remove it:** `claude mcp remove lovable` (add `-s user` if you added it user-wide).
