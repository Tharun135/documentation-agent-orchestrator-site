# Documentation Agent Orchestrator

**Stop AI from inventing documentation.**

Documentation Agent Orchestrator is a VS Code extension that helps technical writers use AI without hallucinations by enforcing strict governance rules.

It documents exactly what you provide, preserves ambiguity explicitly, or stops and asks questions when invention would be required.

## The Problem

When AI rewrites documentation, the real risk is not grammar or tone. It is **silent meaning change**.

- Invented features
- Implied steps that do not exist
- Assumed workflows
- Changed terminology

These mistakes are easy to miss and hard to defend later.

## What This Tool Does

- **Enforces strict governance** on AI documentation rewrites
- **Forces AI to preserve ambiguity** instead of guessing
- **Allows questions only** when documentation would require invention
- **Makes uncertainty explicit** instead of hiding it in fluent text

## What This Tool Deliberately Does NOT Do

Documentation Agent Orchestrator is **not a writing assistant**.

- It does not polish sentences
- It does not improve wording or tone
- It does not simplify language
- It does not fill in missing details

If you want AI to write better, this is not the right tool. If you want AI output you can trust, it is.

## Installation

Install the VS Code extension from the Visual Studio Code Marketplace:

🔗 [Install the VS Code Extension](https://marketplace.visualstudio.com/manage/publishers/tharunsebastian)

## Repository

View the source code and contribute on GitHub:

🔗 [GitHub Repository](https://github.com/Tharun135/documentation-agent-orchestrator-site)

## Documentation Site

For more information, visit:

🔗 [Documentation Agent Orchestrator Website](https://tharun135.github.io/documentation-agent-orchestrator-site/)

## Use Cases

### When to Use This Tool

- You need AI to rewrite technical documentation without changing meaning
- You want to preserve explicit ambiguity in docs
- You need AI to ask questions instead of guessing
- You require governance over AI documentation edits

### When NOT to Use This Tool

- You want AI to improve your writing style
- You need help generating new content from scratch
- You want AI to fill in gaps or make assumptions
- You're looking for a general-purpose writing assistant

## How It Works

The orchestrator acts as a governance layer between you and AI documentation tools. It:

1. Takes your source documentation
2. Applies strict rules to prevent hallucination
3. Requires AI to preserve ambiguity explicitly
4. Forces AI to ask questions when documentation requires invention
5. Returns output that is factually identical to the input

## Philosophy

**AI is fluent. Documentation needs to be correct.**

This tool prioritizes correctness over fluency. It treats documentation as a technical artifact that must preserve meaning exactly, even when that means preserving unclear or ambiguous language.

## License

See the repository for license information.

## Contributing

Contributions are welcome! Please visit the [GitHub repository](https://github.com/Tharun135/documentation-agent-orchestrator-site) to report issues or submit pull requests.
