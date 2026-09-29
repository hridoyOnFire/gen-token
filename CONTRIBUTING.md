# Contributing

Thank you for your interest in contributing! This project is a full-stack USDC transfer demo on Arc Testnet.

## Getting Started

1. Fork the repository and clone your fork
2. Follow the [Quick Start](README.md#quick-start) steps to get the app running locally
3. Create a feature branch: `git checkout -b feat/your-feature`
4. Make your changes
5. Run checks: `bun run check`
6. Commit with a clear message (see below)
7. Open a pull request against `main`

## Commit Messages

Follow the [Conventional Commits](https://www.conventionalcommits.org) format:

```
<type>(<scope>): <short description>

feat(frontend): add reverse transfer button
fix(server): handle missing entity secret gracefully
docs(python): update CLI README with faucet link
chore(deps): bump @circle-fin/developer-controlled-wallets
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

## Code Style

- **TypeScript**: run `bun run check` — zero errors required before merging
- **Python**: follow PEP 8; use type hints; test with `python -m pytest` if adding tests
- **Solidity**: run `forge build` and `forge test` — all tests must pass

## Pull Request Guidelines

- Keep PRs focused — one feature or fix per PR
- Update `README.md` if your change affects usage or the API surface
- Add a short description of what changed and why
- Reference any related issue with `Closes #N`

## Reporting Issues

Use the GitHub issue templates:
- **Bug Report** — unexpected behavior or errors
- **Feature Request** — new capability or improvement

## Security

Do not open public issues for security vulnerabilities. Contact the maintainers privately instead.  
Never include API keys, entity secrets, or wallet addresses in issues or PRs.
