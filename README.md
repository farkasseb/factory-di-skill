# Factory DI Skill

AI skill reference for Swift Factory DI library (hmlongco/Factory) — correct imports, scoping, @MainActor patterns, testing, and common LLM mistakes.

**This skill is slightly opinionated — it favors `FactoryKit` over the legacy `Factory` import, explicit scope specification over defaults, and `ContainerTrait`-based testing over manual `register`/`reset` patterns. These opinions align with Factory v2.5+ best practices, but older codebases may use different conventions.**

## Why this exists

Factory is the most popular Swift DI library, but AI models consistently get its API wrong. They suggest `import Factory` (legacy), miss the `@MainActor` double-annotation requirement, misunderstand scope behavior, and generate test code that breaks under parallel execution.

These aren't subtle issues — they produce code that compiles but fails at runtime or silently uses the wrong isolation context.

### What AI models get wrong vs. what this skill corrects

| Topic | What AI says (wrong) | What's actually correct |
|-------|---------------------|------------------------|
| Import | `import Factory` | `import FactoryKit` (Factory is legacy, causes parallel test issues) |
| @MainActor | Annotates only the closure | Must annotate BOTH computed property AND closure |
| `reset()` | Uses bare `reset()` | Bare reset is nuclear — always specify `.scope` or `.registration` |
| Singletons | Assumes per-container | Singletons use global `Scope.singleton` cache, shared across all containers |
| `@Injected` in tests | Uses property wrapper | Property wrappers bypass test overrides — call `container.type()` directly |
| Swift Testing | Uses `setUp`/`tearDown` | Swift Testing runs tests in parallel — use `ContainerTrait` for isolation |
| `@Observable` | Wraps in `@InjectedObject` | `@Observable` doesn't need `@InjectedObject` — resolve inline |

## Installation

### Claude Code (as a skill)

```bash
mkdir -p ~/.claude/skills/factory-di
cp SKILL.md ~/.claude/skills/factory-di/
cp -r references ~/.claude/skills/factory-di/
```

The skill activates automatically when your code imports Factory/FactoryKit or uses `@Injected`, `Container`, scopes, etc.

### Codex CLI (as agents)

```bash
mkdir -p ~/.agents/skills/factory-di
cp SKILL.md ~/.agents/skills/factory-di/
cp -r references ~/.agents/skills/factory-di/
```

### Other AI tools (as context)

The files are plain markdown — feed them as context to any AI coding assistant:

- **Common mistakes & quick reference**: `SKILL.md`
- **Container wiring & scopes**: `references/wiring.md`
- **Design patterns**: `references/patterns.md`
- **SwiftUI integration**: `references/swiftui.md`
- **Testing (XCTest & Swift Testing)**: `references/testing.md`

## File structure

```
SKILL.md                      # Router + critical LLM mistakes + scope reference + decision trees
references/
├── wiring.md                 # Container setup, extensions, scopes, promised deps, auto-registration
├── patterns.md               # Composition root, facade, decorator, multi-container, modular DI
├── swiftui.md                # @Injected, @InjectedObject, @Observable, previews, environment bridge
└── testing.md                # XCTest setup, Swift Testing with ContainerTrait, parallel isolation
```

## Sources

- [hmlongco/Factory](https://github.com/hmlongco/Factory) — official repository & documentation
- [Factory v2.5 release notes](https://github.com/hmlongco/Factory/releases) — ContainerTrait, FactoryKit module
- Factory source code — verified API signatures, scope behavior, and container internals

## Contributing

Found an inaccuracy, a missing Factory pattern, or have experience with Factory in large-scale projects? PRs welcome. Please include the source (Factory docs, source code reference, or your own production experience) for any additions.
