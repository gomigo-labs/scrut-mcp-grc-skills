# Scrut MCP — GRC Skills

Portable [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that turn the **Scrut MCP server** into finished governance, risk, and compliance (GRC) jobs — file evidence, brief leadership, answer a security questionnaire, and fix a failing cloud test — from whatever AI coding tool you already use.

Each skill is a self-contained folder with a `SKILL.md`. Your agent loads it on the right trigger and runs the Scrut MCP tools in the right order, so you get a named task ("run my compliance digest") instead of a bag of raw tools.

## Skills

| Skill | What it does | Say something like |
|---|---|---|
| [`scrut-upload-evidence`](skills/scrut-upload-evidence/) | Uploads a file (or attaches a link) to the correct evidence item, with a note. File uploads need an agent that can run an HTTP PUT (e.g. `curl`). | "File this access review as evidence." |
| [`scrut-compliance-digest`](skills/scrut-compliance-digest/) | Summarizes program state — frameworks, controls, policies, evidence, tests — for leadership. | "Give me a compliance digest for the board." |
| [`scrut-answer-questionnaire`](skills/scrut-answer-questionnaire/) | Drafts questionnaire answers from your documented posture, with citations; flags gaps. | "Answer this vendor security questionnaire." |
| [`scrut-fix-test`](skills/scrut-fix-test/) | Finds a failing cloud test, pulls Terraform / CloudFormation / AWS CLI remediation, and helps fix your IaC; verify by re-running the test in Scrut (MCP re-runs coming in a future update). | "Fix our failing S3 public-access test." |

## Prerequisite: connect the Scrut MCP server

These skills call the tools exposed by the Scrut MCP server, so you need it connected in your AI client first. The skills themselves are client-agnostic — they work anywhere the Scrut MCP is available.

Use the URL for the region your Scrut account is hosted in (ask your Scrut admin if unsure). Every region uses the same public OAuth Client ID, and you log in with OAuth on first connect — no API keys, no client secret.

| Region | MCP URL |
|---|---|
| US | `https://mcp.us.scrut.io/mcp` |
| India (also the default host) | `https://mcp.scrut.io/mcp` |
| EU | `https://mcp.eu.scrut.io/mcp` |
| AU | `https://mcp.au.scrut.io/mcp` |

**OAuth Client ID (all regions):** `vpubmKiuV6hrcQLBwMLyo4aVWgheoH8U`

- **Claude Code** (OAuth callback port must be `8787`; `--scope user` makes Scrut available in every project, matching skills installed in `~/.claude/skills`):
  ```bash
  claude mcp add --transport http --scope user scrut https://mcp.us.scrut.io/mcp \
    --client-id vpubmKiuV6hrcQLBwMLyo4aVWgheoH8U \
    --callback-port 8787
  ```
- **Claude Desktop / Claude.ai:** Settings → Connectors → Add custom connector. Name `Scrut`, your regional URL, and under Advanced the OAuth Client ID above (leave Client Secret empty). Connect, log in, pick your organization.
- **Cursor:** merge into the `mcpServers` object in `~/.cursor/mcp.json`:
  ```json
  { "mcpServers": { "scrut": { "url": "https://mcp.us.scrut.io/mcp", "auth": { "CLIENT_ID": "vpubmKiuV6hrcQLBwMLyo4aVWgheoH8U" } } } }
  ```
- **Codex CLI:** set `mcp_oauth_callback_port = 8788` in `~/.codex/config.toml`, then:
  ```bash
  codex mcp add scrut --url https://mcp.us.scrut.io/mcp \
    --oauth-client-id vpubmKiuV6hrcQLBwMLyo4aVWgheoH8U
  codex mcp login scrut
  ```
  If the login page's authorize URL has no `resource` parameter, add `--oauth-resource https://mcp.us.scrut.io/mcp` (same URL) to `codex mcp add`.

Swap in your region's URL in each example. **Several workspaces?** Ask your agent to list your Scrut workspaces; the skills pass the chosen `tenant_id` on every call. A workspace must be reached through its own region's URL.

## Install the skills

Skills are just folders — install them wherever your agent looks for skills.

- **Claude Code:** copy a skill folder into `~/.claude/skills/` (e.g. `~/.claude/skills/scrut-upload-evidence/`), or into a project's `.claude/skills/`, then restart.
- **Claude Desktop / Claude.ai:** zip a skill folder and upload it under Settings → Capabilities → Skills.
- **Cursor and other agents:** point the agent at this repo's [`skills/`](skills/) folder, or copy the folder you want into your project.

Every `SKILL.md` is self-contained and readable by any agent — no build step, no runtime.

## Versioning

Skill names are stable; the version lives in each `SKILL.md` frontmatter (`metadata.version`) with a changelog at the bottom of the file. Deeper capabilities bump the version — the folder name never changes, so your installs and references keep working.

## Compatibility and tool availability

The skills target the current Scrut MCP tool set:

| Skill | Version | Scrut MCP tools it calls |
|---|---|---|
| `scrut-upload-evidence` | 2.0.0 | `scrut_search_documents`, `scrut_list_evidence`, `scrut_list_controls`, `scrut_get_control_by_id`, `scrut_get_evidence_by_id`, `scrut_upload_evidence` |
| `scrut-compliance-digest` | 2.0.0 | `scrut_list_frameworks`, `scrut_list_controls`, `scrut_list_policies`, `scrut_list_evidence`, `scrut_list_tests` |
| `scrut-answer-questionnaire` | 1.1.0 | `scrut_answer_question` |
| `scrut-fix-test` | 2.0.0 | `scrut_list_tests`, `scrut_get_test_by_id`, `scrut_answer_question` |

Any skill may also call `scrut_list_workspaces` or `scrut_list_entities` to pick a workspace or entity.

The 2.0.0 skills need the current tool names. Earlier Scrut MCP versions exposed `scrut_upload_file`, `scrut_attach_evidence_document`, and `scrut_get_test`, which have since been replaced. **If you installed a 1.x skill, replace its folder** — 1.x upload, digest, and fix-test skills call tools or fields that no longer exist. Your client lists the tools it currently has. If a tool a skill needs is missing, the skill tells you rather than guessing.
