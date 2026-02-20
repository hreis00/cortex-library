# Cortex Library

A curated collection of AI agents, instructions, prompts, and skills for consistent, high-quality development across repositories and technology stacks.

## Structure

```
cortex-library/
├── AGENTS.md                                  ← AI agent guide for this repo
├── README.md
└── .github/
    ├── agents/                                ← custom agent mode definitions
    │   └── code-reviewer.agent.md
    ├── instructions/                          ← always-on coding rules
    │   └── global-standards.instructions.md
    ├── prompts/                               ← reusable parameterized prompts
    │   └── pr-description.prompt.md
    └── skills/                               ← on-demand domain knowledge
        └── git-conventional-commits/
            └── SKILL.md
```

## Contents

### Agents

| File                                                                             | Purpose                                                                                                                                       |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| [`.github/agents/code-reviewer.agent.md`](.github/agents/code-reviewer.agent.md) | Code review agent. Structured five-pass review covering correctness, security (OWASP), tests, maintainability, and design. Language-agnostic. |

### Instructions

| File                                                                                                             | Scope     | Purpose                                                                                                       |
| ---------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------- |
| [`.github/instructions/global-standards.instructions.md`](.github/instructions/global-standards.instructions.md) | All files | Universal rules for naming, functions, security, error handling, testing, version control, and documentation. |

### Prompts

| File                                                                                   | Purpose                                                                                                      |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [`.github/prompts/pr-description.prompt.md`](.github/prompts/pr-description.prompt.md) | Generates a structured PR description (What/Why/How/Testing/Breaking Changes) from a change summary or diff. |

### Skills

| Skill                                                                                                  | Purpose                                                                                                    |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| [`.github/skills/git-conventional-commits/SKILL.md`](.github/skills/git-conventional-commits/SKILL.md) | Conventional Commits format, branch naming, feature branch workflow, squashing, reverting, and CI tooling. |

## Using These Files

### In a Project

Copy or symlink the files you need into `.github/` in your project:

```bash
# Link all instructions to your project
mkdir -p .github/instructions
ln -s ~/path/to/cortex-library/.github/instructions/global-standards.instructions.md \
      .github/instructions/global-standards.instructions.md

# Link a skill
mkdir -p .github/skills/git-conventional-commits
ln -s ~/path/to/cortex-library/.github/skills/git-conventional-commits/SKILL.md \
      .github/skills/git-conventional-commits/SKILL.md
```

### As a Git Submodule

```bash
git submodule add https://github.com/hreis00/cortex-library .github/cortex-library
```

Then symlink individual files into the expected VS Code paths:

```bash
# Example: activate the global standards instruction
mkdir -p .github/instructions
ln -s .github/cortex-library/.github/instructions/global-standards.instructions.md \
      .github/instructions/global-standards.instructions.md
```

## Contributing

See [`AGENTS.md`](AGENTS.md) for the full contribution guide, file conventions, and quality standards.

## License

MIT — use freely across your projects.
