# Domain Wiki

Auto-maintained knowledge base for platform services. Pages are generated and updated by the `wiki-delta` agent (in the `pr-agents` monorepo) on PR open events.

## Structure

```
wiki/
├── expenses/    — expense tracking service domain
└── approvals/   — expense approval workflow domain
```

## How it works

1. A PR merges to `main` in a service repo.
2. `wiki-agent` reads the diff and opens a PR here with updated markdown.
3. A human reviews and merges the wiki PR.

Every generated page carries an `AI-GENERATED` header. Review before relying on it.

## Contributing

Manual edits are welcome. If you add a new subdomain directory, let the team know so the agent's routing prompt can be updated.
