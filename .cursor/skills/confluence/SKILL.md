---
name: confluence-manager
description: Creates or updates Confluence pages to document project architecture, version upgrades, or technical specifications.
---

Documentation, architectural decision records (ADRs), or runbooks
This skill utilizes the **Atlassian Remote MCP Server** tools when available to interact directly with your Knowledge Base.

## 🧠 Analysis Strategy (Context Extraction)
Before creating a page, analyze the context to extract:
1.  **Space Key:** Identify the target Confluence Space (e.g., `ENG`, `DATA`, `PROJ`). If unknown, ask the user.
2.  **Parent Page:** Determine if this page should sit under a specific parent (e.g., "Technical Specs" or "Runbooks").
3.  **Page Title:** Create a unique, descriptive title. (Format: `[Project] - {Topic}`).
4.  **Content Structure:** Plan the hierarchy (Overview -> Technical Details -> Configuration -> FAQ).

---

## 🛠️ Tool Usage (Atlassian MCP)

If `atlassian` or `confluence` tools are available, **YOU MUST USE THEM**.

### 1. Creating/Updating Pages
**Tool:** `mcp__atlassian__createConfluencePage` / `mcp__atlassian__updateConfluencePage`
* **SpaceId / SpaceKey:** Mandatory.
* **Body Format:** Provide content in **Storage Format** (XHTML) if the tool requires it, OR **Markdown** if the MCP server supports automatic conversion (preferred).
* **Action:** If a page with the same title might exist, try to `search` or `get` it first to avoid duplicates.

### 2. Reading Context
**Tool:** `mcp__atlassian__getConfluencePage`
* Use this to fetch the "Parent Page" content or related architecture docs before writing new ones.

---

## 📝 Output Template (Fallback / Draft Mode)

If tools are **NOT** active, generate this Markdown block ready for copy-pasting into Confluence (using the "Insert Markup" feature):

```markdown
# {Page Title}

**Status:** DRAFT / IN-REVIEW
**Owner:** {User Name}
**Date:** {YYYY-MM-DD}

## 📄 Overview
{Executive summary of the document/feature.}

## 🏗️ Technical Architecture
{Describe the implementation details. Use diagrams if provided via Mermaid.}

## ⚙️ Configuration & Parameters
| Parameter | Type | Description | Default |
| :--- | :--- | :--- | :--- |
| `param_name` | `String` | Description here | `default` |

## 🧪 Testing & Validation
* [ ] Test Case 1
* [ ] Test Case 2

## 🔗 References
* [Link to Jira Ticket]
* [Link to Github PR]
```

## 🚀 Execution Rules
Macro Usage: If generating Markdown for manual copy-paste, suggest using Confluence Macros:

{toc} for Table of Contents.

{info} for notes/warnings.

{code} for code blocks.

No "Lorem Ipsum": Never generate placeholder text. Extract real information from the provided code files.

Linkage: Always try to cross-reference related Jira tickets (using the jira.prompt.md context if available).