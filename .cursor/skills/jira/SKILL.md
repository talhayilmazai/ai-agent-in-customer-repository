---
name: jira-manager
description: Interacts with Jira to create, update, or query tickets based on project tasks and implementation details.
---

## 🎯 Goal
Create, update, or analyze Jira issues based on the current code context, `TODO` comments, or feature implementation status.
This skill utilizes the **Atlassian Remote MCP Server** tools when available.

## 🧠 Analysis Strategy (Context Extraction)
Before creating a task, analyze the active file or diff to extract:
1.  **Summary:** precise title summarizing the work (e.g., "Fix NPE in `SeasonalityDataPrep`").
2.  **Description:** Technical details, reproduction steps (for bugs), or acceptance criteria (for stories).
3.  **Components:** Infer from folder structure (e.g., `rocks_extension/noob` -> `Noob`, `dags/` -> `Airflow`).
4.  **Labels:** Suggest standard labels: `tech-debt`, `bug`, `feature`, `documentation`.

---

## 🛠️ Tool Usage (Atlassian MCP)

If the `atlassian` or `jira` tools are available in the context, **YOU MUST USE THEM** instead of just printing text.

### 1. Creating Issues
**Tool:** `mcp__atlassian__createJiraIssue` (or equivalent provided tool)
* **Project:** Default to the user's active project key (ask if unknown).
* **Description Format:** Use **Markdown**. The MCP server handles Markdown-to-ADF conversion. Do **NOT** try to generate JSON ADF structures manually.

### 2. Updating Issues
**Tool:** `mcp__atlassian__editJiraIssue` / `mcp__atlassian__postComment`
* Trigger this when the user says "Update ticket X" or "Add comment to X".
* Always reference the commit hash or PR link in the comment.

---

## 📝 Output Template (Fallback / Draft Mode)

If tools are **NOT** active, or if the user asks for a draft, generate this Markdown block:

```markdown
### 🎫 Proposed Jira Ticket

**Project:** `[PROJECT-KEY]`
**Type:** `Task` / `Bug` / `Story`
**Summary:** {Concise Title}

**Description:**
{1-2 paragraphs describing the task. Include 'Why' and 'How'.}

**Acceptance Criteria:**
- [ ] Criteria 1
- [ ] Criteria 2

**Technical Notes:**
- Affected Files: `{file_list}`
- Config Changes: `{config_changes}`

**Labels:** `rocks-extension`, `{module_name}`
```

## 🚀 Execution Rules
No Hallucinated IDs: Never invent a Jira Ticket ID (e.g., PROJ-123). If you created one via tool, use the returned ID.

Fail Safe: If the MCP tool fails, immediately provide the "Output Template" content so the user can copy-paste it manually.

Traceability: In the description, always link back to the file path in the repository (e.g., [src/rocks/noob/config.yml]).