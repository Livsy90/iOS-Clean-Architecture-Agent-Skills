# iOS Clean Architecture Agent Skills

<p align="center">
  <img width="320" src="https://github.com/user-attachments/assets/dc2f28eb-61f9-4fe1-8186-a3b2628a573d" />
</p>

A pragmatic Agent Skill for designing, reviewing, and simplifying Clean Architecture in Swift and iOS applications.

It helps AI coding agents reason about dependency direction, composition roots, Use Cases, Repositories, model boundaries, dependency lifetimes, feature assembly, and Swift module design without reproducing textbook layers mechanically.

> Use concrete types by default. Introduce protocols only when a current and explicit requirement justifies them.

The goal is to build the simplest architecture that preserves meaningful boundaries.

## What this skill helps with

Use `clean-architecture-ios` when working on:

- Clean Architecture composition for an iOS feature or application;
- dependency direction between Presentation, Application, Domain, Data, and Infrastructure;
- composition roots, application containers, feature factories, and navigation composition;
- dependency injection without protocol-first design;
- deciding whether a Use Case, Repository, Data Source, mapper, or module adds value;
- identifying thin pass-through layers and speculative abstractions;
- DTO, persistence, domain, and presentation model boundaries;
- external SDK isolation;
- application-, session-, and feature-scoped dependency lifetimes;
- SwiftUI ownership, previews, and test composition;
- logical boundaries and Swift Package modularization.

## Core principles

### Prefer concrete types by default

Dependency injection does not require protocols. Start with concrete structs, classes, actors, values, functions, and closures.

Introduce a protocol when a current requirement needs one, such as:

- multiple implementations that already exist;
- a known second implementation;
- a real module boundary requiring dependency inversion;
- a stable public or integration contract;
- explicit live, offline, cached, platform, or test implementations;
- an external capability that cannot be isolated more simply.

Do not create protocols only because another implementation may exist someday, every dependency is expected to have a mock, or a naming convention expects `Protocol` and `Default`.

### Treat layers as optional tools

A simple feature may use:

```text
View
  ↓
ViewModel
  ↓
Concrete service or client
```

A feature with meaningful data-access policy may use:

```text
View
  ↓
ViewModel
  ↓
Repository
  ↓
API, database, cache, or SDK
```

A feature with application policy or coordination may use:

```text
View
  ↓
ViewModel
  ↓
Use Case
  ↓
Repository or service
```

Use only the layers required by the problem.

### Define boundaries before folders

Clean Architecture is primarily about dependency direction and ownership, not directory names.

Start with logical boundaries. Introduce separate Swift packages only when compile-time enforcement, ownership, visibility, reuse, build performance, or team structure justifies the additional complexity.

## Installation

### Skills CLI

Install the skill with one command:

```bash
npx skills add Livsy90/iOS-Clean-Architecture-Agent-Skills --all
```

Or select it explicitly:

```bash
npx skills add Livsy90/iOS-Clean-Architecture-Agent-Skills \
  --skill clean-architecture-ios
```

The CLI will ask which supported agents should receive the skill and whether to install it for the current project or globally.

This method requires Node.js and `npx`.

### Claude Code

Install the repository as a Claude Code plugin:

```text
/plugin install Livsy90/iOS-Clean-Architecture-Agent-Skills
```

### Manual installation

Clone the repository:

```bash
git clone https://github.com/Livsy90/iOS-Clean-Architecture-Agent-Skills.git
```

For a repository-scoped installation, copy or symlink the skill into:

```text
<repository>/.agents/skills/clean-architecture-ios/
```

For a user-level installation, use:

```text
~/.agents/skills/clean-architecture-ios/
```

Example:

```bash
mkdir -p ~/.agents/skills

cp -R iOS-Clean-Architecture-Agent-Skills/clean-architecture-ios \
  ~/.agents/skills/
```

A symlink is convenient when developing or updating the skill locally:

```bash
ln -s /path/to/iOS-Clean-Architecture-Agent-Skills/clean-architecture-ios \
  ~/.agents/skills/clean-architecture-ios
```

## Skill structure

```text
clean-architecture-ios/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── use-cases-and-repositories.md
    ├── composition-root.md
    └── boundaries-and-models.md
```

### `SKILL.md`

Contains the activation boundaries, architecture workflow, concrete-first policy, decision rules, common gotchas, output expectations, validation requirements, and reference routing.

### `references/use-cases-and-repositories.md`

Read when deciding whether Use Cases, Repositories, Repository protocols, services, or Data Sources add meaningful policy or only forward calls.

### `references/composition-root.md`

Read when building application containers, feature factories, navigation composition, dependency lifetimes, authenticated session graphs, SwiftUI previews, or test graphs.

### `references/boundaries-and-models.md`

Read when separating DTOs, persistence models, domain models, view data, external SDKs, logical layers, or Swift module boundaries.

## Example prompts

```text
Review this iOS feature and tell me whether its Use Cases and Repositories add value.
```

```text
Design a composition root for this SwiftUI application without introducing protocols by default.
```

```text
Should this Repository be a protocol, or can it remain a concrete type?
```

```text
Review this dependency graph and identify thin pass-through layers.
```

```text
Should these DTO, SwiftData, domain, and view models remain separate?
```

```text
How should authenticated and unauthenticated dependency graphs be composed?
```

```text
Should this feature become a separate Swift package?
```

## Expected behavior

The skill should:

1. inspect the existing dependency graph and constraints;
2. identify the user-facing operation;
3. propose the smallest useful composition;
4. preserve correct dependency direction;
5. keep concrete dependencies concrete unless abstraction is justified;
6. explain every introduced protocol or layer;
7. identify unnecessary Use Cases, Repositories, Data Sources, mappers, or modules;
8. define the composition root and important dependency lifetimes;
9. include a minimal Swift example when useful;
10. discuss testing, migration cost, and trade-offs.

For focused questions, the response should stay focused rather than returning a complete architecture report.

## Non-goals

This skill does not enforce:

- a protocol for every dependency;
- a Use Case for every ViewModel action;
- a Repository for every endpoint;
- one model or mapper per layer;
- one Swift package per architectural layer;
- a global dependency container or service locator;
- a particular dependency-injection framework;
- one architecture suitable for every project.

## Repository structure

```text
iOS-Clean-Architecture-Agent-Skills/
├── .claude-plugin/
│   └── plugin.json
├── clean-architecture-ios/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── use-cases-and-repositories.md
│       ├── composition-root.md
│       └── boundaries-and-models.md
├── gemini-extension.json
├── LICENSE
└── README.md
```

## Updating

When installed through the Skills CLI, run the installation command again to update the skill.

For a cloned repository, pull the latest changes:

```bash
git pull
```

A symlinked installation uses the updated checkout automatically.

## License

MIT
