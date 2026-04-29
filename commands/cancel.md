---
description: Cancel an active background Codex job in this repository
argument-hint: '[job-id]'
disable-model-invocation: false
allowed-tools: Bash(node *codex-companion.mjs* cancel*)
---

!`node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" cancel "$ARGUMENTS"`
