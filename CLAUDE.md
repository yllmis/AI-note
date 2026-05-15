# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Obsidian Vault for a Go backend developer preparing for technical interviews. Contains study notes in Chinese covering CS fundamentals, Go ecosystem, backend infrastructure, architecture, projects, algorithms, and AI agents.

**Goal**: Build an AI-powered knowledge system on top of Obsidian that pushes related articles and knowledge blogs to email/QQ/WeChat/Feishu based on recently studied topics.

## Vault Architecture

Numbered folder system with increasing abstraction levels:

| Folder | Purpose |
|--------|---------|
| `00-Inbox` | Quick notes, fleeting thoughts |
| `10-Computer-Science` | OS, networking, fundamentals |
| `20-Go-Ecosystem` | Go language core, concurrency, GC |
| `30-Backend-Infra` | gRPC, middleware (MySQL/Redis/Kafka), scenario questions |
| `40-Architecture` | System design, high-frequency Q&A |
| `50-Projects` | Real projects: FlexiRAG Engine, IM platform, NLP competition |
| `60-AI-Agent` | Frontier exploration |
| `70-LeetCode` | Algorithm practice |
| `80-Life-Log` | Self-intro, review |

## Note Conventions

- **Frontmatter**: YAML with `tags` (array) and `related_project` (YAML array of wikilinks)
- **Tags line**: `标签：#Tag1 #Tag2` at the top of notes without frontmatter
- **Internal links**: `[[folder/note]]` — always use full folder paths in wikilinks; `related_project` must be a YAML array, not a space-separated string (putting multiple `[[link]]` in one string causes Obsidian to create broken directories with `]]` in the name)
- **Q&A format**: Each knowledge point has `**答案**` (answer) and `**记忆**` (mnemonic) — a concise phrase for quick recall
- **Language**: All notes are in Chinese

## Obsidian Plugins

- **Dataview** (0.5.68): Query notes as data with DQL
- **Excalidraw** (2.22.0): Visual diagrams and sketches
- **Smart Connections** (4.3.0): Local AI — uses `bge-micro-v2` embeddings via transformers, Ollama for chat, XML-structured context templates. Config at `.smart-env/smart_env.json`
- **Code Styler** (1.1.7): Code block styling

## Smart Connections Configuration

- Embedding: `TaylorAI/bge-micro-v2` via local transformers adapter
- Chat: Ollama adapter (local LLM)
- Smart blocks: min 200 chars, embed enabled
- Smart sources: min 200 chars, exclude "Untitled" files
- Context template: `xml_structured` preset with `<item>` / `<context>` tags

## Key Design Decisions for AI Push System

The target system needs to:
1. Track recently studied/modified notes (leverage `.smart-env/` embedding data and Obsidian file timestamps)
2. Match related external articles/blogs based on note content and tags
3. Push recommendations to messaging platforms (email, QQ, WeChat, Feishu)

Relevant integration points:
- Smart Connections embeddings already compute similarity — can reuse for article matching
- Dataview queries can filter notes by modification time and tags
- Frontmatter tags and `标签` lines provide topic signals for article retrieval
