---
name: finance:chat
description: Open a conversational session to explore budget strategies and what-if scenarios based on your latest budget snapshot. Run /finance:analyze first to generate a fresh snapshot.
allowed-tools: ["Read"]
---

# Finance Chat

Start an interactive financial strategy session using the latest budget snapshot.

## Before starting

Read `budget-snapshot.md` from the current working directory. Also read `finance.config.md`.

If `budget-snapshot.md` doesn't exist: "No snapshot found. Run `/finance:analyze` first to generate one." Stop here.

If the snapshot's Generated date is more than 6 weeks ago: "Note: your snapshot is from [date] — it may not reflect your current spending. Consider running `/finance:analyze` with fresh statements first. Continuing with existing data..."

## Activate the chat agent

Load the `finance-chat-agent` agent and hand off the conversation with the snapshot and config already loaded as context.
