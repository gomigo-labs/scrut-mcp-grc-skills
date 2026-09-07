# Scrut MCP — GRC Skills

Portable [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that turn the **Scrut MCP server** into finished governance, risk, and compliance (GRC) jobs — file evidence, brief leadership, answer a security questionnaire, and fix a failing cloud test — from whatever AI coding tool you already use.

Each skill is a self-contained folder with a `SKILL.md`. Your agent loads it on the right trigger and runs the Scrut MCP tools in the right order, so you get a named task ("run my compliance digest") instead of a bag of raw tools.

## Skills

| Skill | What it does | Say something like |
|---|---|---|
| [`scrut-upload-evidence`](skills/scrut-upload-evidence/) | Uploads a file (or attaches a link) to the correct evidence item, with a note. | "File this access review as evidence." |
| [`scrut-compliance-digest`](skills/scrut-compliance-digest/) | Summarizes program state — frameworks, controls, policies, evidence, tests — for leadership. | "Give me a compliance digest for the board." |
| [`scrut-answer-questionnaire`](skills/scrut-answer-questionnaire/) | Drafts questionnaire answers from your documented posture, with citations; flags gaps. | "Answer this vendor security questionnaire." |
| [`scrut-fix-test`](skills/scrut-fix-test/) | Finds a failing cloud test — including ones that are failing but ignored — checks what a fix would break before proposing it, helps fix your IaC, and verifies the exposure is gone; confirm by re-running the test in Scrut (MCP re-runs coming in a future update). | "Fix our failing S3 public-access test." |
| [`scrut-find-tests`](skills/scrut-find-tests/) | Locates the right tests in a large library without missing whole categories to pagination. | "Is there a test for S3 encryption?" |

## Working with a large test library

Test libraries get big — a few hundred to a few thousand tests. `scrut_list_tests`
returns them in a fixed order (cloud/CSPM tests **last**) with a default page of
50 and a cap of 200, so a naive call reads a biased slice and can miss entire
categories. On a measured 714-test org, a default call returned 50 vendor
access-review rows and **zero** cloud tests.

Install [`scrut-find-tests`](skills/scrut-find-tests/) alongside the others if
you ask questions about tests — it teaches the agent to search by keyword, take
counts cheaply, and page to completion when it matters.

[**Full findings and the server-side fixes they call for →**](docs/test-volume-findings.md)

## Prerequisite: connect the Scrut MCP server

These skills call the tools exposed by the Scrut MCP server, so you need it connected in your AI client first. The skills themselves are client-agnostic — they work anywhere the Scrut MCP is available.

> **Connecting the Scrut MCP — coming soon.**
> You'll connect using your region's MCP URL and an OAuth Client ID from Scrut (OAuth login on first connect — no API keys, no client secret). Setup steps will be linked here once general availability ships; until then, see the official Scrut documentation or ask your Scrut contact for the current connection details.

## Install the skills

Skills are just folders — install them wherever your agent looks for skills.

- **Claude Code / Claude Desktop:** copy a skill folder into your skills directory, e.g. `~/.claude/skills/scrut-upload-evidence/`, then restart the client.
- **Cursor and other agents:** point the agent at this repo's [`skills/`](skills/) folder, or copy the folder you want into your project.

Every `SKILL.md` is self-contained and readable by any agent — no build step, no runtime.

## Versioning

Skill names are stable; the version lives in each `SKILL.md` frontmatter (`version:`) with a changelog at the bottom of the file. Deeper capabilities bump the version — the folder name never changes, so your installs and references keep working.

## A note on tool availability

Skills use the Scrut MCP's tools. If your connection is on an older version, a tool a skill relies on may not appear yet — your client lists the tools it currently has. Skills degrade gracefully and tell you when a needed tool is unavailable.
