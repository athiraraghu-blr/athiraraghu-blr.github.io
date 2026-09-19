Title: Kiro Unpacked: A Practical Guide to AWS's Spec-Driven IDE and CLI, and What Happened to the 500 Credits
Date: 2026-09-19
Category: Article
Tags: Kiro, Kiro IDE, Kiro CLI, AWS, AI Coding, Agentic IDE, Spec-Driven Development, Kiro Pricing, Kiro Credits, Developer Tools
Slug: kiro-ide-cli-guide-credits-explained-500-to-50

# **What is Kiro?**

1. Kiro is AWS's agentic development environment.

2. Its core idea is **spec-driven development**. Before writing code, the AI turns your prompt into structured requirements, a design, and a task list.

3. Kiro says specs help agents build the right thing instead of guessing, which is how it positions itself against "vibe coding" tools like Cursor and Windsurf.

4. It comes in several interfaces that share **one subscription and one credit pool:**

 **Kiro IDE**: the desktop editor, now at version 1.x.

 **Kiro CLI**: the terminal agent, and the successor to the Q Developer CLI.

 **Kiro Web, Mobile, and Crew**: cloud-based surfaces. Kiro Web uses the same credit model, with no separate cloud compute charge.

# **Kiro IDE: complete overview**

**Core features**

1. **Specs**: requirements, design, and tasks are generated and tracked as files in your project, so the agent follows a plan.

2. **Steering files**: Markdown files in .kiro/steering/ that give the agent standing project rules such as conventions, stack choices, and architecture.

3. **Hooks**: automations that fire on events.

 In IDE 1.0 they use a structured JSON format stored in .kiro/hooks/.

 You can create them by describing what you want in natural language.

4. **MCP support**: connect external tools and data sources.

5. **Custom agents**: Markdown-based agent profiles with tag-based tools (read, write, shell, web), inline MCP servers, and inline permissions.

6. **Capability-based permissions:**

 The agent asks for approval before doing anything you haven't explicitly allowed.

 By default it can read workspace files and run read-only git commands. Everything else prompts.

7. **Agent Focus Mode (experimental):**

 An agent-first layout that can run several parallel sessions.

 Includes Spec, Plan, Bug Fix, and Quick Spec workflows.

8. **Also included**: Powers, agent skills, checkpoints and rewind, compaction for long sessions, dockable chat, and session export.

**Typical workflow**

1. Download the IDE and sign in with GitHub, Google, or an AWS Builder ID.

2. Open a project and add steering files describing your stack and rules.

3. Start a spec, review the generated requirements, design, and tasks, then let the agent execute them task by task.

4. Add hooks (for example, "update tests when a file is saved") and MCP servers as needed.
Watch the credit meter as you go.

# **Kiro CLI: complete overview**

1. Kiro CLI puts the same agent in your terminal.

2. It can write, review, and modify code and automate workflows, either locally or in cloud sessions that persist when you disconnect.

3. Install on macOS, Linux, or Windows:

bash

    curl -fsSL https://cli.kiro.dev/install | bash

**What you can do with it**

1. **Interactive chat** in the terminal, with slash commands such as /model, /save, /load, /usage, and /help, plus ! to run shell commands.

2. **Headless and CI/CD use**. Kiro's own example runs kiro --print "Look at the latest CI failure logs, find the root cause, and apply a fix." inside a script that then commits and opens a pull request.

3. **Custom agents, steering, hooks, and MCP servers**. The CLI reads steering files and MCP configuration from your project's .kiro/ folder, the same context the IDE and Web use.

4. **Cloud sessions.**

 Start with --cloud, add repositories with --repo, and resume with --resume-id <session-id>.

 Pick the task up later from the IDE, web, or phone.

5. **ACP support**, so it works with editors such as JetBrains IDEs, Eclipse, and Zed.

**CLI 3.0 (opt-in early release)**

1. Try it with kiro-cli --v3. It runs alongside your existing 2.x install.

2. **New in 3.0:**

 A built-in Spec agent (/spec new <name>).

 A Plan mode (Shift+Tab).

 /tangent, which branches side-conversations that inherit your context.

 permissions.yaml for structured, capability-based permission rules.

 Standalone hook files and Markdown-based agent configs.

3. **Migration costs to know about:**

 The aws_tool was removed. Use MCP servers for AWS access instead.

 The session format is not backward-compatible, so back up ~/.kiro/sessions/ before upgrading.

 Hooks and permissions need manual migration.

 It does not run on Amazon Linux 2.

**CLI pricing**

1. The CLI has no separate pricing. It is included in the standard Kiro tiers.

# **The credit system**

1. A credit is Kiro's unit of work.

 Simple prompts can cost less than one credit.

 More complex work, such as executing a spec task, usually costs more than one.

 Credits are metered to two decimal places, so the minimum charge is 0.01 credits.

2. What consumes credits: any prompt to the agent (vibe or spec), spec refinement, task execution, and agent hook execution. Usage from the IDE, CLI, and Web all draws from the same pool.

**Plans**

1. **Free**: $0 per month, 50 credits. Access to open-weight models and Claude Sonnet 4.5, with rate limits.

2. **Pro**: $20 per month, 1,000 credits. Premium models.

3. **Pro+**: $40 per month, 2,000 credits. Premium models.

4. **Pro Max**: $100 per month, 5,000 credits. Premium models.

5. **Power**: $200 per month, 10,000 credits. Premium models.

6. **Enterprise**: custom pricing, billed through AWS.

7. In-plan credits work out to roughly $0.02 each on every paid tier.

**Buying more credits**

1. **Individual paid users** can buy add-on credit packs at **$0.04 per credit.**

 Packs range from $5 (125 credits) up to $100.

 Purchased credits roll over and expire 12 months after purchase.

2. **Enterprise admins** can enable overages instead. They are billed at $0.04 per credit at month-end and are off by default.

3. **Free-tier users cannot buy add-on credits.**

**Resets and sharing**

1. Monthly limits reset at the start of each billing month.

2. Unused plan credits **do not roll over.**

3. Each developer needs their own subscription. Enterprise adds consolidated billing, SSO, and usage analytics.

# **Credit consumption: what actually drains your balance**

1. Kiro does not publish a fixed per-action price or a token-to-credit rate.

2. Cost depends on three things:

 the prompt's complexity;

 the model you choose;

 how much the model "thinks" (reasoning effort).

3. Every model has a multiplier relative to **Auto**, Kiro's default router (1.0x).

**Model multipliers**

1. Qwen3 Coder Next: 0.05x

2. MiniMax M2.1: 0.15x

3. DeepSeek 3.2 and MiniMax M2.5: 0.25x

4. GLM-5: 0.5x

5. Claude Haiku 4.5: 0.4x

6. Auto: 1.0x (baseline)

7. GPT-5.6 Luna: 1.1x

8. Claude Sonnet 5, 4.6, and 4.5: 1.3x

9. Claude Opus 5, 4.8, 4.7, and 4.6: 2.2x

10. GPT-5.6 Terra: 2.2x

11. GPT-5.6 Sol: 4.4x

12. GPT-5.6 rates double for requests above 272K tokens.

**How to read these numbers**

1. Kiro's own example: a task costing 10 credits on Auto would cost about 22 on Opus, 4 on Haiku, or 0.5 on Qwen3 Coder Next.

2. Models with the same multiplier won't necessarily cost the same. Output length, thinking depth, and tokenizer differences all change the real cost.

3. Higher reasoning effort (/effort in the CLI) uses more tokens and therefore more credits.

**Practical ways to save credits**

1. Start with Auto, and switch to Opus only when you hit a wall.

2. Use cheap open-weight models for routine work, then a stronger model for the hard steps.

3. Write specific first prompts. A vague opener means burning many more credits course-correcting.

4. Add steering files up front so the agent doesn't rediscover your project every time.

5. Check the usage dashboard regularly. Usage updates at least every five minutes.

6. Scope each request to one task instead of asking for broad rewrites.

# **The "500 to 50" credit change, explained**

The headline is only partly accurate. Here is what actually happened.

**Before May 8, 2026**

1. First-time users got **500 bonus credits**, usable within 14 days.

2. That was on top of the perpetual Free tier of **50 credits**.

**On May 8, 2026**

1. New sign-ups **stopped receiving the 500 bonus credits**.

2. Instead, anyone who upgrades to a paid plan for the first time gets **$20 credited toward the subscription**.

 Kiro describes this as 1,000 credits of usage, double the old 500.

 It requires a valid credit card.

 It requires sign-in via social login or AWS Builder ID, not AWS Identity Center or third-party identity providers.

3. Kiro's stated reasons:

 500 credits often wasn't enough to build something meaningful and form a real opinion.

 Open-weight models have matured enough that free users can get good results without frontier models.

**What did not change**

1. Kiro stated that the existing Free tier remains unchanged. The 50 monthly credits were never cut.

2. Users who signed up before May 8 and hadn't used their original 500 credits can keep using them.

**What it means in practice**

1. A brand-new free account now effectively has 50 credits per month. Before, it had 500 for two weeks plus 50 per month.

2. Users noticed. A GitHub issue from May 10 reports that new accounts get only 50 credits and no 500 bonus.

3. The free tier stretches further than "50" suggests if you stick to low-multiplier models, but it is rate limited and has a weekly quota.

4. Two ways to read the change:

 **Generous reading**: a bigger, more useful trial for people who are serious about evaluating Kiro.

 **Skeptical reading**: it moves evaluation from a free trial to a credit-card-backed paid trial.

# **Advantages**

1. **Structure over guesswork**. Specs, steering, and hooks make agent output traceable and repeatable, which suits teams and larger projects.

2. **One subscription, many surfaces**. The IDE, CLI, Web, and ACP editors share a credit pool and .kiro configuration.

3. **Wide model choice**. Claude, GPT-5.6, and open-weight models are available, and Auto balances cost against quality.

4. **Fractional billing**. Small edits can cost far less than one credit, charged in 0.01 increments.

5. **A usable free tier**. No credit card is needed, and the cheap models make 50 credits go further than they look.

6. **Enterprise features**. SSO through IAM Identity Center, usage analytics, and governance controls, which suit AWS-based organizations.

7. **Published pricing**. One reviewer called full price transparency a rarity in this category.

8. **Fine-grained control**. Capability-based permissions and per-project agents keep the AI inside boundaries you set.

# **Disadvantages**

1. **Opaque credit costs.**

 There is no per-action table and no published token-to-credit rate.

 Critics say the interface can fail to warn you before a large spec-driven loop consumes a big chunk of your allocation.

2. **The free-tier squeeze.**

 The 500-credit trial is gone for new users.

 Premium models are paywalled, and free usage has rate limits and weekly quotas.

3. **No rollover and no sharing**. Plan credits vanish monthly, and every developer needs their own seat.

4. **Heavier for quick prototyping.**

 One review says it feels heavy for pure rapid prototyping and that Cursor is usually faster to start.

 Long sessions can get expensive.

 The spec workflow pays off only if your team actually engages with it.

5. **Migration friction.**

 IDE 1.0 retired inline chat and requires session and hook migration.

 CLI 3.0 has breaking changes, and its sessions can't be resumed in v2.

6. **Restrictions and regional limits.**

 Premium-model availability varies by country and region.

 The free tier is not available in Enterprise or AWS GovCloud.

 GovCloud pricing is roughly 20% higher.

 Kiro prohibits using subscriptions through third-party automation harnesses such as OpenClaw.

7. **Proprietary license**. Both the IDE and CLI are distributed under a standard proprietary license.

8. **A history of billing hiccups**. In 2025, AWS acknowledged a bug that made some tasks consume multiple requests and drained limits faster than expected. It was fixed, but it shows how sensitive credit metering can be.

# **Who should use Kiro?**

1. **Choose Kiro if:**

 you build production software on a team;

 you want enforced structure, steering, and hooks;

 you want one agent across IDE, terminal, cloud, and CI;

 you already work in the AWS ecosystem.

2. **Consider alternatives if:**

 you mostly want fast inline edits;

 you need pooled credits across many people;

 your usage is unpredictable and bursty.

3. **Best way to evaluate:**

 Start on Free.

 Use the first-upgrade bonus for a month on Pro.

 Watch the per-prompt credit display, and size your tier from measured usage rather than the credit numbers on the pricing page.

# **Bottom line**

1. Kiro is one of the more opinionated AI coding tools, and its specs, steering, and hooks reward disciplined teams.

2. The credit system is fair in principle but hard to predict.

3. The "500 to 50" story is about the end of a one-time trial bonus. It is not a cut to the monthly free allowance.

4. Pick a cheap default model, write clear prompts, and keep an eye on the dashboard.

# **Sources and further reading**

1. Kiro pricing and FAQ: kiro.dev/pricing

2. Kiro models and credit multipliers: kiro.dev/docs/models

3. Kiro CLI overview and docs: kiro.dev/cli

4. Kiro IDE 1.0 release notes: kiro.dev/docs/ide/whats-new-v1

5. Kiro CLI 3.0 release notes: kiro.dev/docs/cli/v3

6. Kiro announcement of the May 8, 2026 sign-up bonus change: kiro.dev/blog/new-paid-tier-bonus




