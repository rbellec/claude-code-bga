# Contributing

This repository documents a specific experiment. Contributions that extend the method or apply it to new contexts are welcome.

## Ways to contribute

### Apply the method to another game

The most valuable contribution is **replication**: use the skill file and workflow on a different BGA game, then report what worked, what broke, and what you had to adapt.

1. Fork this repo
2. Copy `SKILL.md` to your Claude Code skills directory
3. Follow the workflow on your own BGA game
4. Document your experience (a new `experiments/your-game/` directory)
5. Submit a PR with your findings

### Improve the skill file

If you discover new BGA framework pitfalls, useful patterns, or corrections to existing entries:

1. Open an issue describing the problem and fix
2. Submit a PR to `SKILL.md` with the change
3. Include the error message, root cause, and fix — not just the fix

### Add prompt templates

If you developed prompts for a game phase not covered (e.g., AI opponent, multiplayer sync, game options):

1. Add a new file in `prompts/` following the existing naming convention
2. Include context about when and why the prompt is used
3. Note any game-specific assumptions that would need adapting

## Guidelines

- **English only** in all files
- **Be specific** — "this error happens when X" is more useful than "there might be issues with Y"
- **Be honest** — document failures and limitations, not just successes
- **Keep the tone technical** — this is an engineering artifact, not a product demo

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
