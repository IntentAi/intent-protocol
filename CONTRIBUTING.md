# Contributing to Intent Protocol

This repo is specification only — no implementation code. Getting it right matters more than getting it done fast.

## Before You Start

Check the [issues](https://github.com/IntentAi/intent-protocol/issues). Pay attention to **blocking relationships** on the right side — some specs depend on others being finalized first. If you find a gap or inconsistency, open an issue before submitting a fix, especially for anything touching wire format or protocol behavior.

## Branching

Work is grouped by phase. Branch off the current phase branch (check the repo or ask a maintainer):

```
git checkout <phase-branch> && git pull origin <phase-branch>
git checkout -b <phase-branch>/8-rate-limit-headers
```

No phase branch? Use `feat/<issue>-description` off `dev`. PR against the **phase branch**, not dev or main.

## Writing Specs

- Be precise. Vague specs lead to incompatible implementations.
- Include request/response examples with realistic data.
- Document error cases, not just happy paths.
- Reference RFCs where applicable.
- Implementation-agnostic. Describe the protocol, not how to code it.
- snake_case for all field names. No emojis.

## Commits

Use `docs` as the commit type. Reference the issue. Explain what changed and why — protocol changes ripple into every implementation.

```
docs(gateway): define reconnection backoff intervals [refs #8]

Specified exponential backoff with jitter for reconnection.
Base 1s, max 60s, full jitter. Matches existing server behavior.
```

## Review

Expect scrutiny on PRs. Every spec change affects the server, web client, and both bot SDKs.

## License

MIT
