---
name: deploy-check
description: Pre-deployment safety verification
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---
Runs in a forked, read-only context to avoid polluting session memory.
