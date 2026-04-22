# AGENTS.md Protocol: Central Command

> **The Operating System for Autonomous AI Agents.**
>
> A curated collection of immutable contracts, architectural constraints, and phase-aware protocols designed to standardize software development across the `suradet-ps` ecosystem.

![Status](https://img.shields.io/badge/Status-Active-success)
![Projects](https://img.shields.io/badge/Total_Projects-9-blue)

---

## Naming Convention & Workflow

Files in this repository follow a strict naming convention to denote **Order of Execution** and **Current Status**. This ensures agents process tasks logically and avoid conflicts.

**Format:** `[Sequence]-[Status]-[Slug].md`

### 1. Status Tags

| Tag | Meaning | Action Required |
| :--- | :--- | :--- |
| **`TODO`** | Pending | Ready to be executed by an AI Agent. |
| **`DONE`** | Completed | Archived context. Use for reference only. Do not execute. |

### 2. Usage (The Bootstrap Prompt)

To activate an agent for a specific task, copy the content of the target `.md` file and prepend this instruction:

> **SYSTEM PROMPT:**
> "You are an autonomous developer agent. Read the attached `AGENTS.md` file. This file is a **BINDING CONTRACT**. You must strictly follow the constraints, stack, and phase-aware workflow defined within. Do not deviate. Confirm your understanding of the **Primary Objective** before generating code."

---

## The Protocol Standard (Anatomy of an Agent)

Every `AGENTS.md` file in this repository strictly adheres to the following structure to ensure consistency:

1.  **Context & Role:** Defines *who* the AI is (e.g., "Senior Rust Architect", "Vue 3 Specialist").
2.  **Immutable Constraints:** Facts that cannot be changed (e.g., "Use Vue 3 Composition API", "No jQuery").
3.  **Phase-Aware Workflow:** Steps that must be done in order (Analysis → Approval → Code → Verify).
4.  **Negative Constraints:** What is explicitly *forbidden* (e.g., "No `any` type", "No direct DOM manipulation").
5.  **Quality Gates:** Tests/Linters that must pass for the task to be considered "Done".

---

## Contributing & Extension

### Adding a New Project
1.  Create a folder named after the project in `kebab-case`.
2.  Add a `README.md` inside that folder describing the project stack and goals.
3.  Create agent files starting with `01-TODO-...`.

### Adding a New Task
1.  Check the highest existing sequence number in the project folder.
2.  Create `[Next_Number]-TODO-[task-name].md`.
3.  Ensure the task is atomic and self-contained.

---

<div align="center">
<i>"Control the Agent, Control the Code."</i>
<br>
<b>Maintained by suradet-ps</b>
</div>
