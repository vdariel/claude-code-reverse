# Claude Code v2.1.88 Source Code Analysis Report

Hello, I’m applek, the owner of a software website.  
Today I obtained a leaked map file, and this project has already been parsed into a report.

Additionally, I’ve also written how to build a third-party client. This project is intended for learning the functionality of Claude Code—please do not use it for illegal purposes.

---

## Table of Contents

1. Overview  
2. Program Startup Flow  
3. Core Architecture  
4. Module Details  
5. API Communication and Data Flow  
6. Tool System  
7. Permissions and Security System  
8. UI Rendering System  
9. Extension Mechanisms  
10. Key Data Flow Diagrams  
11. Build System and Runtime Testing  

---

## 1. Overview

Claude Code is Anthropic’s official CLI programming assistant, developed in TypeScript and using React Ink for terminal UI rendering. The entire program is bundled into a single `cli.js` file (13MB), and the source code includes **1906 internal source files** and 2850 third-party dependency files.

### Tech Stack

- Language: TypeScript  
- Runtime: Node.js (built with Bun)  
- Terminal UI: React Ink (custom fork)  
- State Management: Zustand  
- API Client: @anthropic-ai/sdk  
- CLI Framework: Commander.js  
- Search Engine: ripgrep  
- Protocol Extension: MCP  

---

## 2. Program Startup Flow

The program uses a fast-path startup design, routing early based on CLI arguments to avoid loading unnecessary modules.

---

## 3. Core Architecture

User Input → Main Loop → API Client → Tool Execution → UI Rendering

---

## 4. Module Details

Includes entrypoints, bootstrap state, screens, commands, coordinator, remote sessions, and tasks.

---

## 5. API Communication and Data Flow

Supports multiple providers and uses streaming responses with retry logic and prompt caching.

---

## 6. Tool System

Includes file tools, execution tools, search tools, agent tools, and task management tools.

---

## 7. Permissions and Security System

Multi-layer permission checks with sandboxing and Bash command analysis.

---

## 8. UI Rendering System

Built with React Ink, includes message list, input box, permission dialogs, and background task panels.

---

## 9. Extension Mechanisms

Supports MCP, Skills, Plugins, and Hooks.

---

## 10. Key Data Flow Diagrams

Describes full request-response cycles and agent execution flows.

---

## 11. Build System and Runtime Testing

Built using Bun bundler, compiled into a single executable file.

---

*Generated on 2026-03-31*
