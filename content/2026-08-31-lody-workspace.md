Title: One Chat to Command Them All: Inside Lody, the Workspace for Running AI Coding Agents in Parallel
Date: 2026-08-31
Category: Article
Tags: Lody, AICodingAgents, DeveloperTools, ClaudeCode, GitWorktrees, TeamWorkflow, SoftwareDevelopment, AIAgents
Slug: lody-the-workspace-for-running-ai-coding-agents

If you've ever juggled multiple AI coding assistants across different machines, repos, or operating systems, you know the friction: separate terminals, separate contexts, no shared history, and no easy way for your team to see what an agent is actually doing. Lody is built to remove that friction. It's a team workspace designed specifically for running AI coding agents — not replacing them.

# **What Is Lody?**

Lody is a coordination layer for AI coding agents. Rather than building its own model, it plugs into the agents people already use — Claude Code, Codex, Grok, Kimi, and any agent that speaks the Agent Client Protocol (ACP), including Cursor, Gemini CLI, Cline, Goose, OpenCode, and Qwen. The idea is to keep using the subscriptions and logins you already pay for, but give every agent a shared home where sessions, diffs, and decisions live together.

It's available as a web app, desktop app, and mobile app, so a session started on a laptop can be checked on and steered from a phone.

# *Core Features*

**Parallel worktrees**. Sessions run in isolated Git worktrees, so multiple agents can work on the same repository at the same time without stepping on each other's changes.

**Live diff review**. Every turn or full session produces a diff that sits right next to the conversation, so you can review what an agent changed without leaving the chat and add line-level comments directly on the code.

**Agent-to-agent delegation**. One conversation can act as a coordinator, spinning up sub-sessions to test on different platforms, hand off a review-then-fix loop between two agents, or patch a broken dependency in a separate session and pull the fix back into the main one.

**Remote machine access**. Running npx lody daemon start on a server, cloud VM, or home desktop connects that machine to your workspace. From there, sessions can be created, messaged, and monitored from a laptop, a phone, a script, or CI.

**GitHub integration**. Pull request status, CI checks, and review conversations are surfaced inside the session, so merging doesn't require switching context.

**Team workspace**. Sessions are shared, so decisions stay in the team workspace — teammates can open each other's sessions and keep chatting, while individual machines stay private until explicitly shared. There's also usage tracking that breaks down token and cost usage by model and teammate.

# **Typical Uses**

1. **Running the same fix across environments** — testing a change on macOS, Linux, and Windows simultaneously via sub-agents.

2. **Splitting review and implementation** — one agent reviews code while another implements fixes, iterating without manual handoffs.

3. **Fixing cross-repo dependency issues** — patching a broken dependency in an isolated session, verifying it, then merging the fix back.

4. **Keeping teams in sync** — engineering teams sharing agent sessions so anyone can pick up where another left off.

5. **Mobile oversight** — checking on long-running agent tasks and approving permission prompts from a phone.

6. **Server-based automation** — kicking off agent sessions from CI pipelines or scripts against a persistently connected machine.

# **Who It's For**

Lody is aimed at developers and teams already using one or more AI coding agents who want a shared, reviewable, multi-agent workflow instead of disconnected terminal sessions. It's less useful for someone running a single agent on a single local project with no need for team visibility or cross-machine coordination.