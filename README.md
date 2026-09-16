# Obsidian Knowledge Base

This repository contains a structured **Obsidian knowledge base** designed to be used together with an LLM through the **Obsidian MCP (Model Context Protocol)**.

The goal is to allow an LLM to navigate, search, understand, and work with the knowledge stored in this repository while preserving the relationships between notes through Obsidian links.

---

## What is this repository?

This repository is an **Obsidian Vault** containing Markdown notes organized as a knowledge base.

The notes are intentionally written in a way that allows an LLM to understand not only individual concepts, but also the relationships between them.

For example, a note about an integration may reference concepts such as:

* `[[Move]]`
* `[[Action]]`
* `[[Activity]]`
* `[[Order]]`
* `[[Plan]]`
* `[[Trip]]`
* `[[Receipt]]`
* `[[Vehicle]]`
* `[[Organization]]`

These relationships are an important part of the knowledge base and should be preserved when modifying or adding documentation.

The repository can therefore be used both:

1. Directly as an **Obsidian Vault**.
2. As a knowledge source for an **LLM through MCP**.

---

# Architecture

The intended setup is:

```text
┌─────────────────────┐
│       Obsidian      │
│                     │
│   Markdown Vault    │
└──────────┬──────────┘
           │
           │ MCP
           ▼
┌─────────────────────┐
│    Obsidian MCP     │
│      Server         │
│                     │
│  Search / Read /    │
│  Navigate / Write   │
└──────────┬──────────┘
           │
           │ MCP
           ▼
┌─────────────────────┐
│         LLM         │
│                     │
│ ChatGPT / Claude /  │
│ Cursor / other LLM  │
└─────────────────────┘
```

The LLM should **not be expected to understand the repository by simply receiving individual files**.

Instead, the LLM should interact with the Vault through the MCP server, allowing it to search and navigate the knowledge graph.

---

# Requirements

Before using this repository with an LLM, you need:

### 1. Obsidian

Install [Obsidian](https://obsidian.md/) and open this repository as a Vault.

The repository itself should be opened as the Obsidian Vault root.

### 2. Docker

You need Docker installed and running.

The Obsidian MCP server runs through Docker and exposes the Vault to your LLM using the Model Context Protocol.

Make sure Docker is available from your terminal:

```bash
docker --version
```

and:

```bash
docker ps
```

### 3. An LLM with MCP support

You need an LLM/client capable of connecting to an MCP server.

Examples include clients that support MCP such as:

* ChatGPT
* Claude
* Cursor
* Other MCP-compatible LLM clients

The exact configuration depends on the client you are using.

---

# Getting Started

## 1. Clone the repository

Clone the repository locally:

```bash
git clone <repository-url>
```

Then enter the repository:

```bash
cd <repository-name>
```

---

## 2. Open the repository in Obsidian

Open Obsidian and select:

**Open folder as vault**

Select the cloned repository directory.

You should now be able to navigate the notes and their relationships directly from Obsidian.

---

## 3. Configure the Obsidian MCP server

The MCP server needs access to this Vault.

The general setup is:

```text
LLM
 │
 │ MCP
 ▼
Obsidian MCP Server
 │
 │ filesystem access
 ▼
Obsidian Vault
```

The Docker container must have access to the local directory containing this repository.

Depending on the MCP implementation you use, the configuration will normally involve mounting the Vault into the container.

For example, conceptually:

```text
Local machine

/path/to/obsidian-vault
        │
        │ Docker volume
        ▼
Obsidian MCP container
```

> The exact Docker command/configuration depends on the Obsidian MCP server implementation you choose. Follow that MCP server's setup instructions and make sure the repository directory is mounted and accessible.

---

# Connecting the LLM

Once the MCP server is running, configure your LLM client to connect to it.

The final connection should look like:

```text
                    ┌──────────────────┐
                    │  Obsidian Vault  │
                    │                  │
                    │   *.md files     │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Obsidian MCP    │
                    │      Server      │
                    └────────┬─────────┘
                             │
                             │ MCP
                             ▼
                    ┌──────────────────┐
                    │       LLM        │
                    └──────────────────┘
```

After connecting, the LLM should be able to use the MCP tools to search and read the knowledge base.

---

# How to use the Knowledge Base

Once the MCP connection is working, you can interact with the knowledge base naturally.

For example:

```text
Explain how the Openlane integration works.
```

The LLM can search the relevant notes and follow their Obsidian links.

You can also ask:

```text
What integrations use the Activity Payload?
```

or:

```text
Find everything related to webhooks and explain how they work.
```

or:

```text
Show me the relationship between Move, Action, Plan and Trip.
```

The expected workflow is:

```text
Question
   │
   ▼
LLM searches the Vault
   │
   ▼
Finds relevant notes
   │
   ▼
Follows [[WikiLinks]]
   │
   ▼
Builds context
   │
   ▼
Generates answer
```

---

# Working with the Notes

## Preserve Obsidian links

When creating or modifying notes, preserve existing WikiLinks whenever possible.

Use:

```markdown
[[Organization]]
```

instead of:

```markdown
Organization
```

when the concept already exists as a note.

This allows Obsidian and the LLM to understand the relationship between concepts.

---

## Reuse existing concepts

Before creating a new note, search the Vault to determine whether the concept already exists.

For example, do not create:

```text
Vehicle.md
```

if a `Vehicle` concept already exists.

Instead, reference:

```markdown
[[Vehicle]]
```

This prevents duplicate concepts and keeps the knowledge graph consistent.

---

# Knowledge Base Conventions

The repository uses consistent terminology for the concepts that appear across integrations.

Some important concepts include:

| Concept          | Meaning                                                          |
| ---------------- | ---------------------------------------------------------------- |
| **Move**         | A movement/action represented in the system                      |
| **Action**       | An operation/activity associated with a Move                     |
| **Activity**     | An integration-facing representation of work                     |
| **Order**        | A customer/order-level representation of a movement              |
| **Plan**         | A plan containing one or more movements                          |
| **Trip**         | A planned execution of movements                                 |
| **Receipt**      | Information associated with the completion/receipt of a movement |
| **Vehicle**      | The vehicle involved in the movement                             |
| **Organization** | The organization/account associated with the data                |

These terms may overlap depending on the integration. When documenting a specific integration, preserve the terminology used by that integration while linking back to the canonical concepts where appropriate.

---

# Adding New Knowledge

When adding a new note:

1. Check whether the concept already exists.
2. Reuse existing `[[WikiLinks]]`.
3. Keep the note focused on one concept.
4. Link related concepts.
5. Avoid duplicating information already documented elsewhere.
6. Prefer explicit relationships over repeated explanations.
7. Keep integration-specific behavior inside the corresponding integration notes.

For example:

```markdown
# Openlane Integration

The Openlane integration consumes [[Activity Payload]] events.

An [[Activity]] represents the movement that must be synchronized
with Openlane.

The integration updates the corresponding [[Order]] when the
movement progresses.
```

This is preferable to redefining `Activity`, `Order`, and `Vehicle` inside every integration document.

---

# Using the LLM to Maintain the Vault

One of the main purposes of connecting the LLM through MCP is to use it as a knowledge-base assistant.

You can ask it to:

### Search

```text
Find all notes related to Kinesis.
```

### Explain

```text
Explain how webhook integrations work using the knowledge in this Vault.
```

### Connect concepts

```text
How are Activity Payload, Move and Organization related?
```

### Identify gaps

```text
Find integration concepts that are referenced but don't have their own note.
```

### Improve documentation

```text
Review the Openlane integration documentation and identify missing links
to existing concepts.
```

### Create documentation

```text
Create a note documenting this integration using the conventions already
used in the Vault.
```

When asking the LLM to modify the Vault, it should first search existing notes and conventions before creating new content.

---

# Important: The Repository Is the Source of Truth

The Markdown files in this repository are the knowledge source.

The LLM should **not silently invent missing information**.

If something is not documented, the preferred behavior is to say that the information is not currently available in the knowledge base.

When external information is introduced, it should be clearly distinguished from information coming from the Vault.

---

# Git Workflow

Because the Vault is stored as Markdown files, it can be version-controlled normally with Git.

Before making large changes:

```bash
git status
```

Review changes:

```bash
git diff
```

Then commit:

```bash
git add .
git commit -m "Update integration documentation"
```

This makes it possible to track changes made manually through Obsidian as well as changes made through an LLM.

---

# Recommended Workflow

A good workflow is:

```text
1. Clone repository
       ↓
2. Open repository in Obsidian
       ↓
3. Start/configure Obsidian MCP
       ↓
4. Connect your LLM to MCP
       ↓
5. Ask the LLM to search the Vault
       ↓
6. Review existing knowledge
       ↓
7. Add/update notes
       ↓
8. Review changes in Git
       ↓
9. Commit changes
```

The most important principle is:

> **Search first, reuse existing concepts, then create or modify notes.**

This keeps the knowledge graph coherent as the repository grows.

---

# Troubleshooting

### The LLM cannot find my notes

Check that:

* The repository is opened as an Obsidian Vault.
* Docker is running.
* The MCP server is running.
* The Vault directory is correctly mounted into the MCP container.
* The LLM is connected to the correct MCP server.
* The MCP server has permission to read the Vault.

### The LLM creates duplicate concepts

Ask it to search the Vault before creating a new note:

```text
Before creating anything, search the Vault for existing notes
representing this concept and reuse them if they exist.
```

### WikiLinks are not being followed

Make sure the links use Obsidian's standard syntax:

```markdown
[[Note Name]]
```

and that the referenced note actually exists in the Vault.

---

# Contributing

When contributing to this repository, prioritize:

* Consistent terminology
* Reusable concepts
* Clear relationships between notes
* Obsidian WikiLinks
* Integration-specific documentation
* Avoiding duplicated knowledge
* Keeping the knowledge base understandable to both humans and LLMs

The objective is not to create the largest collection of notes.

The objective is to create a **connected, navigable, and reliable knowledge graph** that both humans and LLMs can use effectively.
