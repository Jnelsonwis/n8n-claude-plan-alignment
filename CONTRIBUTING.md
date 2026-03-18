# Contributing

Contributions are welcome — especially new stack examples, prompt improvements, and bug reports from people running this in production.

## What to Contribute

| Type | Examples |
|------|---------|
| **New stack examples** | Laravel, Spring Boot, Rust/Axum, SvelteKit, NestJS |
| **Prompt improvements** | Better invariant patterns, more precise output formatting |
| **Workflow variants** | No-Gotify version, multi-project version, GitHub Actions trigger |
| **Bug fixes** | SSE parsing edge cases, timeout handling, response parsing |
| **Documentation** | Clearer setup steps, video walkthroughs, translated READMEs |

## Before Opening a PR

1. **Test your prompt** — SSH to your server and run `claude --print "your prompt" --model sonnet --output-format text` directly before wiring it into the workflow. This isolates prompt issues from n8n wiring issues.

2. **Keep stack examples self-contained** — each example in `run-claude-prompt-node-example.md` should work as a drop-in replacement for the `prompt` variable in the Run Claude Code node. Include the `--allowed-tools` line for the stack.

3. **Don't commit credentials** — double-check that no real tokens, URLs, passwords, or hostnames end up in the workflow JSON or examples. Use the `YOUR_*` placeholder convention.

4. **Match the existing output section names** — if your prompt changes the output sections (`## Aligned`, `## Misaligned`, etc.), update the regex patterns in `Build Gotify Message` and `Format Report` accordingly and document the change.

## Reporting Bugs

Use the bug report issue template. The most useful bug reports include:
- n8n version
- Claude model used (`sonnet`, `opus`, etc.)
- Whether the issue is in the SSE connection, the Claude prompt, or response parsing
- The raw output from the Run Claude Code node (redact any sensitive paths)

## Proposing a New Stack Example

Open a "New stack example" issue first if you're unsure whether the stack fits. For well-known frameworks just open a PR directly.

Stack examples live in `examples/run-claude-prompt-node-example.md` under the **Stack-Specific Examples** section. Follow the existing format:
- Section header: `### Framework + ORM + Auth`
- Prompt as a fenced code block
- Numbered checks covering: schema, routes, auth/hooks, file placement, dependencies

## Workflow Variants

If you've adapted this workflow significantly (different trigger, different notification system, multi-project support, etc.), consider adding it as a separate JSON file under `workflow/` with a short description at the top of the file as a comment in the `meta` object.

## Code of Conduct

Be direct and specific in reviews. Focus on the prompt logic and workflow wiring, not style preferences. This is a practical tool — contributions that make it more useful for real projects are the priority.
