# AGENTS.md

## Agent Orchestration

Proactively use available subagents, skills, and specialized tools whenever they improve quality, speed, or reliability. Do not wait for explicit instructions to delegate work.

### Preferred Routing

#### Planning & Coordination

- codebase-orchestrator
- workflow-orchestrator
- task-distributor
- context-manager
- code-mapper

#### Architecture & Code Quality

- reviewer
- code-reviewer
- refactoring-specialist
- technical-writer

#### Debugging

- debugger
- error-detective
- browser-debugger

#### Frontend & UX

- frontend-developer
- react-specialist
- nextjs-developer
- typescript-pro
- javascript-pro
- ui-designer
- ui-fixer
- ui-ux-tester
- accessibility-tester

#### Backend & APIs

- backend-developer
- node-specialist
- api-designer
- api-documenter
- postgres-pro
- sql-pro
- websocket-engineer

#### Testing & Quality

- qa-expert
- test-automator
- performance-engineer
- performance-monitor

#### Infrastructure & Operations

- docker-expert
- deployment-engineer
- terraform-engineer
- kubernetes-specialist
- cloud-architect
- devops-engineer
- sre-engineer

#### AI / MCP

- ai-engineer
- llm-architect
- mcp-developer
- prompt-engineer
- prompt-regression-tester
- eval-engineer
- hallucination-investigator
- knowledge-synthesizer
- search-specialist
- multi-agent-coordinator

### Skill Usage

Automatically use installed Skills whenever they are relevant to the task. Prefer using specialized Skills instead of reinventing workflows.

Examples include:

- Frontend/UI design
- Browser and end-to-end testing
- MCP development
- Documentation
- PDF, DOCX, XLSX, PPTX handling
- Web artifacts
- Theme generation
- shadcn/ui workflows

### Review Workflow

For every non-trivial implementation:

- Perform a complete self-review before considering the task finished.
- Review correctness, maintainability, readability, security, edge cases, performance, accessibility, and test coverage.
- If a dedicated review agent is available, use it automatically.
- If a CodeRabbit-equivalent review is available, treat it as the final review step before completion.

### Git Rules

- Never commit, push, merge, rebase, tag, or open pull requests unless explicitly requested.
- Never rewrite Git history unless explicitly instructed.
- Keep changes focused and logically grouped.
- Prefer Conventional Commits.
- Before any commit, run available formatting, linting, type checking, and tests.
- Summarize changed files, validation results, remaining risks, and suggested follow-up work.

### Safety Rules

- Never expose secrets, credentials, API keys, tokens, or private data.
- Never modify production infrastructure, deployment configuration, or environment files unless explicitly requested.
- Prefer backward-compatible, incremental changes.
- Avoid unnecessary breaking changes.
- Never delete user data or destructive resources without explicit confirmation.
- Preserve the existing architecture unless a better solution is clearly justified.
- Inspect the repository and existing patterns before making significant changes.

### Working Style

For medium and large tasks:

1. Understand the existing codebase and conventions.
2. Create a concise implementation plan.
3. Delegate work to the most appropriate subagents and Skills.
4. Implement incrementally.
5. Continuously validate assumptions.
6. Run formatting, linting, type checking, tests, and other relevant validation.
7. Perform a final review before presenting the result.
8. Return a concise summary of changes, validation results, remaining risks, and recommended next steps.
