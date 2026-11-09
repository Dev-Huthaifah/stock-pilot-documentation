# AI Integration Plan

**Status: Planned; provider, model, architecture, and scope are undecided.**

Candidate capabilities (not yet implemented): natural-language inventory questions, stock/movement summaries, low-stock insights, replenishment suggestions, and product search assistance.

Security requirements:
- Respect authenticated user's company and permissions.
- Never expose another company's data through prompts, retrieval, logs, or tool calls.
- Prefer narrowly scoped backend tools over unrestricted database access.
- Validate proposed write operations and require confirmation where appropriate.
- Keep provider secrets server-side.
- Apply usage limits and privacy-conscious logging.

Decisions required: provider/model, retrieval or embeddings, tool-calling design, tenant filtering, cost controls, privacy, and retention.
