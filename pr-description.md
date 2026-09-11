# Weekly market research: add Trello and NVIDIA CUDA Docs MCP servers

## Summary

Weekly cross-vendor comparison against the AI Registry (`ai.open-vsx.org/api/v1/all.json`) surfaced 4 MCP servers approved by other vendors but not yet by Theia. After filtering out entries that don't fit Theia's use case, two are added here:

- **`com.atlassian/trello-mcp-server`** — Official Atlassian remote MCP server for Trello (boards, lists, cards, checklists, OAuth-authenticated). Complements Theia's existing `com.atlassian/atlassian-mcp-server` (Jira/Confluence) approval and fills a lightweight task/kanban-management gap.
- **`com.nvidia.ngc.nsight.copilot.api/cuda-docs`** — Official, Anthropic-registry-verified NVIDIA server for searching CUDA documentation and code samples. Fills a GPU/CUDA developer-tooling gap.

## Not added

- `com.jetbrains/mcp-server` — exposes IntelliJ-IDE-specific tooling (file editing, debugging, VCS inside JetBrains IDEs); not relevant to Theia users.
- `io.github.NVIDIA/elements` — NVIDIA's UI design system for robotics/autonomous-vehicle work; too narrow/niche for general adoption.

## Removals

None. `npm run validate:local` passed cleanly (no skill-source warnings). Several existing MCP approvals (AWS, Asana, Docker Hub, Brightdata, e2b, Notion, Neon) show a recurring "not found in Anthropic MCP registry" warning, same as the new Trello entry — these are all active, well-known official servers, so this reads as a gap in Anthropic's registry coverage rather than a deprecation signal, and isn't treated as a removal candidate.

## Validation

```
AI_REGISTRY_CORE_DIR=../ai-registry-core npm run validate:local
```
→ `PASSED: All files valid`
