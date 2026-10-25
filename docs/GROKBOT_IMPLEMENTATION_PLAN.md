# Multi-Agent GrokBot Platform — Implementation Plan

## Objective
Build a self-hostable, multi-model AI coding and automation platform with persistent agents, project workspaces, GitHub integration, secure execution, and coordinated multi-agent tasks. Begin as a single-user private beta; design for future SaaS multi-tenancy.

## Product requirements
- Agent creator: name, role, instructions, model, tools, permissions, budgets, project memory.
- Streaming conversations and durable history.
- Model adapters for xAI Grok, Anthropic Claude, and OpenAI; support provider-specific capabilities and usage metering.
- Project workspaces linked to GitHub repositories.
- Controlled tool execution: file reads/writes, search, shell, tests, diffs, pull requests.
- Supervisor agent that delegates bounded tasks to specialized architect, developer, QA, security, design, and research agents.
- Run logs, cancellation, approval checkpoints, retries, resumable checkpoints, and cost ceilings.
- Optional future organization billing with Stripe.

## Architecture
```
Next.js dashboard (Clerk auth)
        |
Next.js API / backend authorization
        |
PostgreSQL + Prisma  <-- pgvector memory
        |
Redis/BullMQ job queue
        |
Agent worker (TypeScript)
  |-- Supervisor / task graph / checkpoints
  |-- Model router: xAI / Anthropic / OpenAI
  |-- Tool registry and permission gate
  |-- GitHub App
  |-- Sandbox manager --> isolated code execution
        |
Object storage for logs, artifacts and snapshots
```

Keep execution workers separate from the Vercel-hosted web application. The worker controls a bounded model/tool loop and emits durable run events. Workers must never execute untrusted agent commands directly on the host.

## Stack
- TypeScript monorepo (pnpm workspaces or Turborepo)
- Next.js, Tailwind, shadcn/ui
- Clerk authentication; organization-aware authorization
- PostgreSQL, Prisma, pgvector
- Redis + BullMQ
- S3-compatible object storage
- GitHub App installation-scoped repository access
- Docker-based sandbox prototype; stronger VM isolation before hosting untrusted tenants
- Vercel for UI/API; separate worker host
- Stripe for later billing

## Suggested monorepo
```
apps/web
apps/worker
packages/database
packages/agent-core
packages/ai-providers
packages/tool-registry
packages/github
packages/sandbox
packages/memory
packages/billing
packages/shared
infrastructure/docker
infrastructure/deployment
tests/integration
tests/e2e
docs
```

## Data model
- organizations(id, name, plan)
- users(id, organization_id, role)
- agents(id, owner_id, model, instructions, config)
- agent_tools(agent_id, tool_id, permissions)
- projects(id, organization_id, repository_id)
- repositories(id, installation_id, repo_name)
- conversations(id, project_id, agent_id)
- messages(id, conversation_id, role, content)
- runs(id, agent_id, status, budget, checkpoint)
- run_steps(id, run_id, type, status, result)
- tool_executions(id, run_id, tool, arguments, result)
- memories(id, scope, content, embedding)
- artifacts(id, run_id, storage_key)
- approvals(id, run_id, action, status)
- usage_events(id, organization_id, model, tokens, cost)
- audit_events(id, actor_id, action, timestamp)

Use tenant scoping on every query, encrypted credentials, and audit trails for sensitive actions.

## Agent runtime contract
1. Validate user, project, tool permissions, model and run budget.
2. Create queued run; worker leases job and records checkpoint.
3. Retrieve relevant conversation/project/agent memory.
4. Call selected model; stream events to the UI.
5. Validate requested tool against registry and per-run policy.
6. If high-risk action, suspend in waiting_approval.
7. Execute allowed tool inside sandbox, save structured result and checkpoint.
8. Repeat with limits on elapsed time, tokens, cost, tool calls and delegation depth.
9. Store final output, artifacts and logs; mark completed/failed/cancelled.
10. Do not blindly replay side-effecting operations after worker crashes.

## Secure coding workflow
- Clone authorized GitHub repo at pinned commit to ephemeral workspace.
- Create job-specific branch.
- Use non-root sandbox, resource limits, isolated filesystem, restricted network egress and secret access.
- Never mount Docker socket in untrusted sandbox.
- Allow only approved tool actions; require approval for deploy, destructive commands, secret access and production changes.
- Run tests/lint; capture diffs and logs; open PR after approval.
- Destroy workspace on completion, with retained artifact snapshots.

## Multi-agent orchestration
Start sequentially:
Architect produces plan -> Developer implements -> QA tests -> Security reviews -> Supervisor summarizes.
Each delegated task has typed input/output, a budget, timeout, and limited tools. Parallel execution only for independent branches/worktrees.

## Milestones
### Week 1: Foundation
Set up monorepo, CI, database, Clerk, organization/project models, dashboard skeleton.
**Acceptance:** authenticated user can create and view projects.

### Week 2: Agent builder and chat
Agent CRUD, instruction editor, model adapters, streaming UI, conversation persistence.
**Acceptance:** user creates two agents and chats with either.

### Week 3: Tool/runtime engine
Durable run state machine, BullMQ worker, tool registry, approval gate, usage events.
**Acceptance:** agent completes a bounded tool task with inspectable logs.

### Week 4: GitHub and sandboxes
GitHub App, repository picker, sandbox lifecycle, file/shell/test tools, branch/PR workflow.
**Acceptance:** agent modifies a test repository, runs tests and proposes PR.

### Week 5: Memory and multi-agent
Project memory, checkpoints, supervisor delegation, sequential agent teams.
**Acceptance:** architect/developer/QA complete a small change with bounded costs.

### Week 6: Private beta hardening
Run cancellation, retries, budgets, security tests, telemetry, deployment docs.
**Acceptance:** repeatable private deployment and documented end-to-end demo.

Public SaaS launch requires additional security, abuse prevention, tenant isolation and billing validation.

## Codex: start here
You are implementing this repository as a production-minded MVP. First inspect existing files and preserve them. Propose the concrete Phase 1 implementation plan and list decisions that need confirmation. Then scaffold the monorepo, Next.js app, Prisma schema, local Docker Compose services, environment variable example, lint/typecheck/test scripts, and GitHub Actions CI. Do not deploy or incur paid API usage without approval. Use placeholders for all secrets. Keep changes on a feature branch and open a PR for review. Provide a concise README with local startup commands and explicit acceptance checks.

### Phase 1 definition of done
- `pnpm install`, lint, typecheck and tests run successfully.
- Local Postgres and Redis start through Compose.
- Prisma migrations apply cleanly.
- Authenticated project dashboard works with basic project CRUD.
- Secrets never committed.
- CI passes and PR explains changes.
