---
name: subagent-creator
description: Use when creating, spinning up, scaffolding, or designing a new subagent from scratch, or when someone says "create a new agent", "build an agent for X", "make a subagent that does Y", "scaffold a helper agent", or wants to design a specialized AI coworker. Also use proactively when the user outlines a complex task that would benefit from a dedicated, isolated subagent workflow.
---

## What This Skill Does

Helps architect, scaffold, and generate fully configured subagents from scratch through a purely conversational workflow. The user describes what they want the subagent to do in plain language. This skill handles the entire setup—translating their goals into a complete, syntactically correct subagent file.

This skill is responsible for the entire structural lifecycle: creating the configuration file, writing the frontmatter metadata, and establishing the subagent's system persona.

For technical reference on valid metadata structures and allowed execution tools, see [reference.md](reference.md).

---

## Core Principle

**The user describes the role. You engineer the subagent.**

Never ask the user to specify configuration keys, frontmatter parameters, or raw metadata fields. Instead of asking "What tools or permissionMode do you want to assign to this subagent?", ask "Should this helper be allowed to modify your files directly, or should it only read them and give advice?"

---

## Interactive Creation Workflow

When a user triggers this skill to create a new subagent, execute this exact progressive disclosure loop to build it:

### 1. The Conversational Intake
Do not dump a massive, overwhelming technical questionnaire. Ask **one or two high-impact questions** maximum to define the subagent's boundaries:
* "What is the single most important task this subagent should focus on?"
* "Should it have permission to execute terminal commands/run code, or should its access be limited to reading and writing files?"

### 2. Behind-the-Scenes Translation
Map their plain-language answers directly to a structured subagent configuration blueprint:
* **Name & Description:** Convert their target role into a lowercase, hyphenated file name (e.g., `api-tester`). Craft a highly descriptive `description` string that acts as the routing prompt so the system knows exactly when to trigger it.
* **Tool Permissions:** If they said *"Keep it safe,"* restrict `tools` to `read`. If they said *"Let it build things,"* include `read`, `write`, and relevant execution tools.

### 3. Clean Execution & Architecture Setup
* Open your environment's file writing tools and create the target subagent file directly at the designated directory path (e.g., `.bob/agents/[agent-name].md`).
* Inject the complete frontmatter configuration block along with its tailored system instructions.
* Present a clean summary of the creation to the user in plain language, showing the file path and providing the exact `@mention` trigger phrase they can use to start chatting with their new subagent immediately.
