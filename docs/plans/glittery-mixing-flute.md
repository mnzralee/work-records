# GX Protocol Multi-Agent Claude Code Architecture Plan

## Executive Summary

This plan establishes a sophisticated multi-agent orchestration system for GX Protocol development. Based on research from Anthropic's official best practices, the Claude Agent SDK, and community patterns, we're implementing a **department-style agent hierarchy** where one orchestrator agent coordinates specialized sub-agents for backend, frontend, architecture, testing, debugging, and more.

---

## Research Findings Summary

### From Anthropic Official Sources
1. **Explore → Plan → Code → Commit** workflow is essential - agents jump to coding without explicit planning
2. **Subagents preserve context** - heavy operations run in isolated windows
3. **Specific instructions** dramatically improve first-attempt success rates
4. **Extended thinking** triggers: "think", "think hard", "think harder", "ultrathink"
5. **Two-agent pattern** for long tasks: Initializer Agent + Coding Agent
6. **State artifacts**: Feature lists (JSON), progress files, descriptive git commits

### From Agent SDK & Community
1. **One agent, one job** - clear inputs/outputs, single goal
2. **Chain agents in pipelines**: Analyst → Architect → Implementer → Tester → Reviewer
3. **Permission minimization** - start from deny-all, allowlist only what's needed
4. **Hooks for orchestration** - SubagentStop, PreToolUse, PostToolUse
5. **Haiku for exploration** (fast, cheap), **Sonnet for coding**, **Opus for architecture**

---

## Current State Analysis

### Existing Assets (Already in Place)
```
.claude/
├── commands/           # Skill files (8 existing)
│   ├── backend.md      # 637 lines - comprehensive backend patterns
│   ├── frontend.md     # 420 lines - wallet/admin patterns
│   ├── chaincode.md    # 375 lines - Hyperledger Fabric patterns
│   ├── infra.md        # 398 lines - K8s deployment patterns
│   ├── review.md       # 257 lines - code review checklists
│   ├── debug.md        # 369 lines - troubleshooting guides
│   ├── commit.md       # Git commit skill
│   └── pr.md           # PR creation skill
└── settings.local.json # 78 permission rules
```

### What's Missing
1. **Subagent definitions** (`.claude/agents/`) - true subagent orchestration
2. **Hooks configuration** - lifecycle automation
3. **Orchestrator agent** - master coordinator
4. **Progress tracking** - cross-session state
5. **Model routing** - optimal model per task type
6. **Pipeline definitions** - sequential agent chains

---

## Architecture Design

### 1. Agent Hierarchy

```
                    ┌─────────────────────┐
                    │   ORCHESTRATOR      │
                    │   (Main Claude)     │
                    │   Model: Opus       │
                    └─────────┬───────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   ARCHITECT   │    │   RESEARCHER  │    │   PLANNER     │
│   Model: Opus │    │   Model: Haiku│    │   Model: Sonnet│
│   Read-only   │    │   Read-only   │    │   Read-only   │
└───────────────┘    └───────────────┘    └───────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION TIER                     │
├───────────────┬───────────────┬───────────────┬───────────┤
│   BACKEND     │   FRONTEND    │   CHAINCODE   │   INFRA   │
│   Sonnet      │   Sonnet      │   Sonnet      │   Sonnet  │
│   Read/Write  │   Read/Write  │   Read/Write  │   Read/Write│
└───────────────┴───────────────┴───────────────┴───────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│                    VALIDATION TIER                         │
├───────────────┬───────────────┬───────────────┬───────────┤
│   TESTER      │   REVIEWER    │   DEBUGGER    │   SECURITY│
│   Sonnet      │   Sonnet      │   Sonnet      │   Opus    │
│   Read + Bash │   Read-only   │   Read + Bash │   Read-only│
└───────────────┴───────────────┴───────────────┴───────────┘
```

### 2. Agent Definitions

#### Core Agents to Create

| Agent | Purpose | Model | Tools | Proactive |
|-------|---------|-------|-------|-----------|
| `architect` | System design, API contracts, module planning | Opus | Read, Grep, Glob | Yes |
| `researcher` | Codebase exploration, pattern discovery | Haiku | Read, Grep, Glob | Yes |
| `backend-impl` | Backend code implementation | Sonnet | Read, Edit, Write, Bash | No |
| `frontend-impl` | Frontend code implementation | Sonnet | Read, Edit, Write, Bash | No |
| `chaincode-impl` | Smart contract implementation | Sonnet | Read, Edit, Write, Bash | No |
| `infra-impl` | K8s manifest and deployment work | Sonnet | Read, Edit, Write, Bash | No |
| `tester` | Write and run tests | Sonnet | Read, Edit, Bash | No |
| `reviewer` | Code review and quality checks | Sonnet | Read, Grep, Glob | Yes |
| `debugger` | Diagnose and fix issues | Sonnet | Read, Grep, Bash | No |
| `security` | Security audit and vulnerability detection | Opus | Read, Grep, Glob | Yes |

### 3. Directory Structure

```
.claude/
├── CLAUDE.md                    # Project context (existing, enhanced)
├── settings.json                # Shared settings
├── settings.local.json          # Personal overrides (existing)
│
├── agents/                      # NEW: Subagent definitions
│   ├── architect/
│   │   └── AGENT.md
│   ├── researcher/
│   │   └── AGENT.md
│   ├── backend-impl/
│   │   └── AGENT.md
│   ├── frontend-impl/
│   │   └── AGENT.md
│   ├── chaincode-impl/
│   │   └── AGENT.md
│   ├── infra-impl/
│   │   └── AGENT.md
│   ├── tester/
│   │   └── AGENT.md
│   ├── reviewer/
│   │   └── AGENT.md
│   ├── debugger/
│   │   └── AGENT.md
│   └── security/
│       └── AGENT.md
│
├── skills/                      # NEW: Reusable skill definitions
│   ├── module-planning/
│   │   └── SKILL.md
│   ├── tdd-workflow/
│   │   └── SKILL.md
│   ├── api-design/
│   │   └── SKILL.md
│   └── verification/
│       └── SKILL.md
│
├── commands/                    # Existing user-invocable commands
│   └── ... (keep existing)
│
├── rules/                       # NEW: Modular instruction rules
│   ├── clean-architecture.md
│   ├── testing-pyramid.md
│   ├── git-workflow.md
│   └── security-standards.md
│
├── hooks/                       # NEW: Lifecycle automation scripts
│   ├── pre-commit-check.sh
│   ├── auto-format.py
│   └── security-scan.sh
│
└── progress/                    # NEW: Cross-session state
    ├── current-module.json
    └── feature-checklist.json
```

---

## Implementation Plan

### Phase 1: Core Agent Definitions (Priority: HIGH)

Create 10 agent definition files in `.claude/agents/`:

#### 1.1 Architect Agent
```markdown
# .claude/agents/architect/AGENT.md
---
name: architect
description: Expert system architect. Use for module planning, API design, and architectural decisions. Invoked proactively for complex features.
tools: Read, Grep, Glob, WebSearch
disallowedTools: Edit, Write, Bash
model: opus
---

# System Architect Agent

You are the lead architect for GX Protocol. Your role is to:
1. Design module structures and API contracts
2. Identify patterns and conventions from the codebase
3. Create implementation plans with file paths and interfaces
4. Ensure architectural consistency across services

## When Invoked
- New module/feature requests
- Before any multi-file implementation
- API design decisions
- Cross-service integration planning

## Output Format
Provide structured plans with:
- File paths to create/modify
- Interface definitions
- Dependency analysis
- Risk assessment
```

#### 1.2 Researcher Agent (Fast Exploration)
```markdown
# .claude/agents/researcher/AGENT.md
---
name: researcher
description: Fast codebase explorer. Use for pattern discovery, file search, and understanding existing code. Runs on Haiku for speed.
tools: Read, Grep, Glob
disallowedTools: Edit, Write, Bash
model: haiku
---

# Codebase Researcher

You are a rapid research agent. Your role is to:
1. Find existing implementations and patterns
2. Locate files matching criteria
3. Understand code structure
4. Report findings concisely

## Focus
- Speed over depth
- Concrete file paths and line numbers
- Pattern identification
- Brief summaries
```

#### 1.3-1.10: Similar Definitions for Other Agents
(Each with appropriate tools, model, and specialized prompts)

### Phase 2: Enhanced Settings & Hooks

#### 2.1 Settings Configuration
```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(npm run:*)",
      "Bash(git:*)",
      "Bash(kubectl get:*)",
      "Bash(kubectl logs:*)",
      "Bash(npx prisma:*)",
      "Bash(npx tsc:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Read(.env)",
      "Read(**/secrets/**)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/auto-format.py"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "tester",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Tests completed, review results'"
          }
        ]
      }
    ]
  },
  "env": {
    "GX_AGENT_MODE": "orchestrated"
  }
}
```

### Phase 3: Workflow Skills

#### 3.1 Module Planning Skill
```markdown
# .claude/skills/module-planning/SKILL.md
---
name: module-planning
description: Plan new module implementation
user-invocable: true
---

# Module Planning Workflow

When planning a new module:

1. **Research Phase** (use researcher agent)
   - Find similar existing modules
   - Identify patterns to follow
   - List relevant files

2. **Architecture Phase** (use architect agent)
   - Design module structure
   - Define interfaces
   - Plan API endpoints
   - Create database schema (if needed)

3. **Create Plan Document**
   Write to `docs/workrecords/MODULE-X-{name}.md`:
   - Module overview
   - Technical specification
   - File list with descriptions
   - Implementation order
   - Testing strategy
```

### Phase 4: Progress Tracking

#### 4.1 Feature Checklist (JSON for reliability)
```json
// .claude/progress/feature-checklist.json
{
  "module": "MODULE-4-ONBOARDING",
  "status": "in_progress",
  "features": [
    {"id": "OB-001", "name": "Welcome screen", "passes": false},
    {"id": "OB-002", "name": "Email verification", "passes": false},
    {"id": "OB-003", "name": "Phone verification", "passes": false}
  ],
  "completedAt": null,
  "lastSession": "2026-01-25"
}
```

---

## Workflow Patterns

### Pattern 1: Full Module Implementation

```
User: "Build the onboarding module"

Orchestrator Actions:
1. Invoke ARCHITECT agent
   - Design module structure
   - Output: Implementation plan

2. Invoke RESEARCHER agent (parallel)
   - Find existing patterns
   - Output: Reference implementations

3. Create feature checklist in .claude/progress/

4. For each feature:
   a. Invoke BACKEND-IMPL or FRONTEND-IMPL
   b. Invoke TESTER to write tests
   c. Invoke REVIEWER to validate
   d. Update checklist

5. Final SECURITY audit

6. Invoke COMMIT skill
```

### Pattern 2: Bug Fix

```
User: "Fix the login issue"

Orchestrator Actions:
1. Invoke DEBUGGER agent
   - Diagnose root cause
   - Output: Issue analysis + file paths

2. Invoke BACKEND-IMPL or FRONTEND-IMPL
   - Apply fix

3. Invoke TESTER
   - Verify fix + add regression test

4. Invoke REVIEWER
   - Ensure quality

5. Invoke COMMIT skill
```

### Pattern 3: Code Review

```
User: "/review PR #123"

Orchestrator Actions:
1. Invoke REVIEWER agent
   - Quality check
   - Output: Findings

2. Invoke SECURITY agent
   - Security audit
   - Output: Vulnerabilities

3. Synthesize and report
```

---

## Configuration Choices (User Confirmed)

| Setting | Choice |
|---------|--------|
| **Rollout** | All 10 agents at once |
| **Model Strategy** | Optimize for capability - Opus (architecture/security), Sonnet (implementation), Haiku (exploration) |
| **Autonomy Level** | High - Orchestrator auto-delegates without confirmation |

---

## Files to Create (All HIGH Priority)

| # | File | Purpose |
|---|------|---------|
| 1 | `.claude/agents/architect/AGENT.md` | System design agent (Opus) |
| 2 | `.claude/agents/researcher/AGENT.md` | Fast exploration agent (Haiku) |
| 3 | `.claude/agents/backend-impl/AGENT.md` | Backend implementation (Sonnet) |
| 4 | `.claude/agents/frontend-impl/AGENT.md` | Frontend implementation (Sonnet) |
| 5 | `.claude/agents/chaincode-impl/AGENT.md` | Smart contract implementation (Sonnet) |
| 6 | `.claude/agents/infra-impl/AGENT.md` | Infrastructure implementation (Sonnet) |
| 7 | `.claude/agents/tester/AGENT.md` | Testing specialist (Sonnet) |
| 8 | `.claude/agents/reviewer/AGENT.md` | Code review (Sonnet) |
| 9 | `.claude/agents/debugger/AGENT.md` | Debugging specialist (Sonnet) |
| 10 | `.claude/agents/security/AGENT.md` | Security audit (Opus) |
| 11 | `.claude/settings.json` | Shared permissions & hooks |
| 12 | `.claude/skills/module-planning/SKILL.md` | Module planning workflow |
| 13 | `.claude/skills/tdd-workflow/SKILL.md` | TDD workflow |
| 14 | `.claude/rules/clean-architecture.md` | Architecture rules |
| 15 | `.claude/hooks/auto-format.py` | Auto-formatting hook |
| 16 | `.claude/progress/current-module.json` | State tracking |

---

## Verification Plan

After implementation, verify by:

1. **Agent Loading**
   - Run `/agents` to see all agents listed
   - Manually invoke each agent to confirm prompts work

2. **Orchestration Test**
   - Request: "Plan a simple feature"
   - Verify: Architect and Researcher agents invoked automatically

3. **Pipeline Test**
   - Request: "Implement a simple API endpoint with tests"
   - Verify: Backend-impl → Tester → Reviewer chain works

4. **Hook Test**
   - Edit a file
   - Verify auto-format runs

---

## Key Principles Applied

1. **One Agent, One Job** - Each agent has clear, focused responsibility
2. **Model Routing** - Opus for architecture, Haiku for exploration, Sonnet for implementation
3. **Permission Minimization** - Agents get only tools they need
4. **Context Isolation** - Subagents prevent context bloat
5. **State Persistence** - JSON artifacts for cross-session continuity
6. **Explicit Planning** - Architecture before implementation

---

## Sources

- [Claude Code Best Practices (Anthropic)](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Building Agents with Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents)
- [Awesome Claude Code (Community)](https://github.com/hesreallyhim/awesome-claude-code)
