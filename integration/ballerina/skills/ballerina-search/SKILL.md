---
name: ballerina-search
description: Search for Ballerina packages, modules, and tools on Ballerina Central. Use when looking for libraries, connectors, or CLI tools for a Ballerina project.
allowed-tools: Bash(bal *) WebFetch WebSearch
---

Find Ballerina packages for the user's needs. **Always prefer official packages** (`ballerina/` or `ballerinax/`) — only suggest community packages if no official alternative exists, and flag them as unofficial.

## Strategy

1. Run `bal search $ARGUMENTS`.
2. If insufficient, fetch `https://central.ballerina.io/search?q=$ARGUMENTS` for richer details.
3. If neither yields results, use `WebSearch` as a fallback.

## Output

For each relevant package: name (org/module), latest version, brief description, and import statement (e.g., `import ballerinax/kafka;`).
