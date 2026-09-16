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

Before u
