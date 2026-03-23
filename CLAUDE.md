# CLAUDE.md

Instructions for Claude when contributing to this repository.

## What This Repo Is

A library of Claude AI skills focused on **TON smart contract security auditing**.

This repo is not a general FunC/Tact development helper. Contributions should sharpen vulnerability detection for TON-specific execution semantics.

## Structure

```
ton-auditor/ # Security review of TON smart contracts while you develop.
CLAUDE.md    # This file (read by Claude Code)
```

## Rules

- Keep guidance specific to TON async execution, bounce logic, gas modes, Jetton flows, and storage packing.
- Prefer real TON failure modes over generic blockchain advice.
- No fabricated examples - outputs must reflect real model responses.
- No secrets, API keys, or personal data.
