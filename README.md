# Frontend Skills

A curated repository of specialized skills and engineering reference guides for modern frontend web development, optimized for AI coding assistants and developers.

## Overview

This repository provides reusable skills designed to guide AI agents and engineering teams through best practices, design patterns, performance optimizations, and system architecture for frontend applications.

## Available Skills

| Skill | Description | Status |
| --- | --- | --- |
| [**react-design-patterns**](skills/react-design-patterns/SKILL.md) | Consolidated playbook of React design patterns covering component composition, state management, async UI, and performance optimizations. | Available |
| [**frontend-system-design**](skills/frontend-system-design/SKILL.md) | Architectural playbooks covering rendering strategies (CSR, SSR, SSG, ISR, RSC, PPR), state taxonomy & caching, real-time/offline sync, DOM virtualization, and Core Web Vitals. | Available |

## Directory Structure

```
frontend-skills/
└── skills/
    ├── react-design-patterns/       # React component & state design patterns
    │   ├── SKILL.md                 # Agent entry point & decision matrix
    │   ├── README.md                # Skill overview
    │   └── references/              # Detailed implementation guides
    └── frontend-system-design/      # Frontend architecture & system design
        ├── SKILL.md                 # Agent entry point & decision matrix
        ├── README.md                # Skill overview
        └── references/              # Detailed implementation guides
```

## Usage

### Installing via `npx skills` CLI (Recommended)

Install skills directly into your project using the `npx skills` CLI:

```bash
# Interactively select and install skills from this repository
npx skills add abhithshaji/frontend-skills

# Install a specific skill directly
npx skills add abhithshaji/frontend-skills --skill react-design-patterns
npx skills add abhithshaji/frontend-skills --skill frontend-system-design
```

### Manual Installation
Alternatively, copy or symlink skills from the `skills/` directory into your project's agent skills directory (e.g., `.agents/skills/` or global agent configuration paths):

```bash
mkdir -p .agents/skills
cp -r skills/react-design-patterns .agents/skills/
cp -r skills/frontend-system-design .agents/skills/
```

### For Developers
Explore the `references/` within each skill folder for detailed code examples, architectural tradeoffs, and best practices.

