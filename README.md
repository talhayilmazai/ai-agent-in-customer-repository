# AI Agent Best Practices & Customer Repository Template

This repository serves as a **Knowledge Base** and **Toolset** designed for AI Agents (such as GitHub Copilot and Cursor) acting as Data Engineers. It provides a structured environment with specialized instructions, skills, and agents to ensure consistent and high-quality code generation and maintenance.

## 📂 Repository Structure

The repository is organized into two main "brain" directories, catering to different AI environments:

### 1. 🤖 `.github/` (Universal / Copilot Brain)
This directory contains the core logic, instructions, and skills that are portable across different AI tools, with a primary focus on GitHub Copilot.

#### `agents/`
Specialized agent definitions that perform complex analysis or multi-step tasks.
- **Dependency Tracker**: Analyzes the "blast radius" of changes. It checks how modifying a table, config, or DAG impacts downstream dependencies.

#### `instructions/`
The "Long-Term Memory" for the AI. These files contain strict guidelines and architectural rules.
- **PySpark**: Best practices for writing performant and clean PySpark code (e.g., using the `rocks` framework).
- **Config**: Rules for managing the configuration-driven architecture (`config.yml`, `config_metadata.yml`).
- **Development**: General coding standards, naming conventions, and workflow guidelines.

#### `skills/`
Executable tools that allow the agent to interact with external systems.
- **Jira**: Create, update, and retrieve Jira tickets based on code context.
- **Confluence**: specific skill to manage documentation on Confluence.
- **GitHub CLI**: Handle Pull Requests, and repository management tasks.
- **Doc Generation**: Automate the creation of technical documentation.
- **Notebook Generation**: Generate test and analysis notebooks for data validation.
- **PR Review**: Guidelines and tools for reviewing code changes.

#### `prompts/`
Reusable prompts for standardized outputs.
- **Changelog Manager**: Generates consistent changelog entries based on git diffs.

---

### 2. 🤖 `.cursor/` (Cursor IDE Brain)
This directory contains specific configurations and enhancements tailored for the **Cursor IPO** (IDE).

#### `rules/` (`.mdc` files)
Context-aware rules that automatically trigger based on the files you are editing.
- **master.mdc**: The **Orchestrator**. It runs on every request to decide which other rules or skills to activate.
- **pyspark.mdc**: Automatically loaded when editing `.py` files, providing PySpark specific context.
- **config.mdc**: Loaded when editing `.yml` config files to enforce schema validity.
- **development.mdc**: General development guidelines.

#### `agents/`
Definitions for Cursor-specific agents that can be invoked within the IDE.

#### `skills/`
Cursor-specific skills that extend the IDE's capabilities.
- **Jira**: Interacts with Jira to create, update, or query tickets.
- **Confluence**: Manages documentation on Confluence.
- **GitHub CLI**: Handles PRs and other repository tasks.
- **Doc Generation**: Automates documentation creation.
- **Notebook Generation**: Creates test and analysis notebooks.
- **PR Review**: Provides guidelines and tools for code reviews.
- **Deslop**: Cleans up code and removes technical debt.

#### `commands/`
Custom commands to streamline workflows.
- **changeLog**: A command to quickly generate or update the changelog.

---

## 🚀 Usage

This repository is intended to be used with AI coding assistants that can read repository context.

### The "Router" Concept
The core logic for the AI's behavior is defined in **[`.github/copilot-instructions.md`](.github/copilot-instructions.md)** (for Copilot) and **[`.cursor/rules/master.mdc`](.cursor/rules/master.mdc)** (for Cursor).

**Decision Protocol:**
1.  **Analyze Request**: The AI determines if the user wants to write code, perform a task, or analyze something.
2.  **Route**:
    *   **Coding?** -> Load `instructions/` or `rules/`.
    *   **Task?** -> Activate a `skill` (e.g., "Create Jira Ticket").
    *   **Analysis?** -> Activate an `agent` (e.g., "Dependency Tracker").
3.  **Execute**: Perform the action using the loaded context.

## ⚙️ Setup & Configuration

### MCP Server Configuration (Cursor)

To enable the Atlassian MCP server in Cursor, add the following configuration to your settings:

```json
{
  "mcpServers": {
    "Atlassian-MCP-Server": {
      "url": "https://mcp.atlassian.com/v1/mcp"
    }
  }
}
```
