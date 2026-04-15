# claude-code-superpowers-guide

Installation
The simplest method is via the official Claude plugin marketplace. In your Claude Code terminal, run:
bash/plugin install superpowers@claude-plugins-official
Alternatively, you can install from the author's marketplace:
bash/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
You'll need Claude Code 2.0.13 or later. GitHub After installation, quit and restart Claude Code. You can verify it worked by running /help — you should see new commands like /superpowers:brainstorm, write-plan, and execute-plan. Medium
How It Works
Superpowers is a free, open-source plugin that installs 14 structured skills into Claude Code, giving it a predefined agentic framework to draw from. MindStudio Once installed, the skills activate automatically — you don't need to remember special commands. Just describe what you want to build, and the workflow kicks in.
The Core Workflow (7 phases)

Brainstorming — Activates before writing code. Refines rough ideas through questions, explores alternatives, and presents design in sections for validation. ClaudePluginHub
Planning — Breaks work into 2–5 minute tasks with exact file paths, exact commands, and complete code. Builder.io
Git worktree isolation — Creates an isolated branch for the feature work.
TDD (Test-Driven Development) — Rigorously applies the RED-GREEN-REFACTOR cycle: write tests first that fail, write the minimum code to pass, then refactor. If Claude writes code before tests, it automatically deletes the code and restarts with tests. Pasquale Pillitteri
Subagent-driven development — Fast iteration with two-stage review: spec compliance first, then code quality. GitHub
Code review — Reviews work against the plan and reports issues by severity. Critical issues block progress until fixed. Medium
Finishing — Verifies tests, presents options to merge, create a PR, keep working, or discard the branch, then cleans up the worktree. Medium

Personal Skills
You can create your own skills in ~/.config/superpowers/skills/. Personal skills take priority over core skills when paths match, allowing you to customize behaviors without modifying the plugin. Betazeta This is where you'd add skills specific to your stack or workflows — for example, a skill for your C++ backup integration development lifecycle.
Quick Start
Just start a new session and describe what you want to build. For example:

"help me plan this feature"
"let's debug this issue"
"I want to build a new module for this project"

The relevant skill activates automatically. The core context is very token-light — fewer than 2k tokens initially, and it pulls in additional skill docs via shell script as needed. 

Example of guide tailored to DB backup system integration workflow: 
The core idea is a two-phase workflow:
Phase 1 — Brainstorming acts as your requirements + gap analysis session. You start by telling Claude Code what DB you want to integrate, and Superpowers prevents it from jumping to code. Instead, it walks you through the DB capability survey, integration pattern selection (XBSA streaming vs CLI wrapper vs SQL command vs file copy), and most importantly — the gap analysis that catches the landmines early (like YashanDB's dropped-tablespace recovery limitation, or Redis 6→7 AOF format break).
Phase 2 — Planning converts the brainstorming decisions into a sequence of 2–5 minute micro-tasks, each with a test-first requirement, exact file paths, and clear acceptance criteria. The plan follows your existing class hierarchy pattern ([DB]BackupClient → [DB]RestoreClient → connection → backup path → restore path → PITR → hardening).
The key discipline: feed it the actual vendor docs during brainstorming rather than letting it guess, and save the brainstorming output as the seed for your requirements.md / design.md references. Fixing a 3-line task description in the plan is infinitely cheaper than reworking 300 lines of C++.