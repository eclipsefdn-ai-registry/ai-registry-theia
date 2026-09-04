# AI Registry Market Research

Compare the Theia IDE MCP server and skill approvals against other vendors in the AI Registry and the broader MCP ecosystem. Suggest up to $ARGUMENTS (default: 3) additions or removals. Suggest 0 if nothing meaningful has changed.

This runs weekly — optimize for low effort. Prefer deterministic checks over manual investigation. Missing a candidate is fine; it'll surface next run.

## Step 1: Cross-vendor comparison (primary source, do this first)

Do these in parallel:

1. Fetch `https://ai.open-vsx.org/api/v1/tools/theia-ide.json` — extract every MCP server ID and skill ID Theia currently approves. Call these `THEIA_SERVERS` and `THEIA_SKILLS`.
2. Fetch `https://ai.open-vsx.org/api/v1/all.json` — extract every MCP server and skill across ALL tools/vendors.

Compare the two lists for both servers and skills. Report anything approved by other tools but NOT by Theia. These are the highest-signal candidates — another vendor already vetted them. This is the main source of addition candidates for both servers and skills — the rest of Step 2 is a cheap bonus pass, not a requirement.

If good gaps are found here, they're usually enough — skip straight to Step 4 unless you want a broader pulse check.

## Step 2: Quick pulse check for new MCP servers (best-effort, MCP only)

There's no comparable ranking source for skills (no equivalent to an MCP leaderboard), so skip skill discovery here — Step 1 is the only realistic source for skill gaps.

Fetch `https://glama.ai/mcp/servers?sortBy=githubStars` and skim the top ~20 for official, first-party servers not in `THEIA_SERVERS`. If the fetch fails or nothing stands out, move on — don't chase secondary sources or dig deeper.

## Step 3: Health check on current approvals (deterministic)

Run the registry's own validator instead of manually checking repos:

```
AI_REGISTRY_CORE_DIR=../ai-registry-core npm run validate:local
```

(Falls back to `npm run validate`, which clones `ai-registry-core` fresh, if that directory isn't available locally.)

This already does the deterministic work:
- **MCP approvals**: looked up live against the Anthropic MCP registry — a `WARNING: ... serverId "..." not found in Anthropic MCP registry` flags a candidate for removal (deprecated/renamed/never-registered).
- **Skill approvals**: each skill source is actually fetched — a `WARNING: ... could not verify skill source` flags a candidate for removal (repo gone, path moved).

Treat any such warning as a removal candidate. Don't separately hand-check GitHub repo existence — the validator already did it.

## Step 4: Produce recommendations

Present a table with max $ARGUMENTS rows:

| Action | Server/Skill ID | Reason (1 sentence) | In Anthropic registry? |
|--------|----------------|----------------------|------------------------|

For additions: state what gap it fills.
For removals: state why (validator warning, deprecated, superseded, redundant).

## Step 5: Apply changes (ask first)

Ask the user: "Should I apply these changes?"

If yes, for each change:
- **Removals**: delete the approval file from `mcp/` or `skills/`.
- **MCP additions**: read `organization.json` and `ai-docs/mcp-approval.md`, then read 1-2 existing files in `mcp/` as format reference. Create the new file in `mcp/` following the naming convention (`/` → `--`). Include `installConfigs` for both `theia-ide` and `theia-ide-next` with identical config. Omit `installUrl` (the org's `mcpInstallUrlPrefix` handles it).
- **Skill additions**: read 1-2 existing files in `skills/` as format reference (the format is simple: `skillId`, `date`, `source.url` plus optional `source.path` — a single path, array of paths, or a `dir/*` glob for multi-skill repos — and `installConfigs` listing `theia-ide` and `theia-ide-next` with no `config` needed). Create the new file in `skills/` following the naming convention (`/` → `--`).

Then run `AI_REGISTRY_CORE_DIR=../ai-registry-core npm run validate:local` (or `npm run validate`) and report the result.

## Step 6: PR description

Save a short summary of changes and reasoning to `pr-description.md` in the repo root.
