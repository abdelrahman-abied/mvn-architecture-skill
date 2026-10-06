# MVN Architect — Claude Code plugin

A Claude Code plugin that teaches Claude to **scaffold, analyze, and refactor Flutter apps** using **Clean Architecture + the MVN (Model-View-Notifier) pattern** with modern Riverpod code generation.

It is the AI companion to the **[Clean Architecture and MVN with RiverPod](https://abdelrahman-abied.github.io/mvn-architecture-pattern/)** VSCode extension: the extension generates the folder/file scaffolding, and this skill makes Claude fill those files with code that respects the exact same layering and naming.

> MVN was introduced by **Abdulrahman Abied** — the bridge between View and Model is a Riverpod `Notifier`, never a "ViewModel" or "Controller".
> See the [pattern overview](https://abdelrahman-abied.github.io/mvn-architecture-pattern/) and the [Medium article](https://medium.com/@abied.abiad/beyond-mvvm-introducing-the-model-view-notifier-mvn-pattern-for-flutter-with-riverpod-2b123e28f26a).

## What it enforces

- **Canonical structure** per feature — `domain/` (entities, repository contracts, usecases), `data/` (request/response models, repository impl, sources), `presentation/` (notifier, state, view, widget). Folder names are **singular** and match the extension verbatim.
- **The dependency rule** — `presentation ➔ domain ⬅ data`; domain is pure Dart.
- **MVN presentation** — sealed state, `Notifier` classes via `@riverpod`, exhaustive `switch`, `ref.listen` for side effects, `.select()` for targeted rebuilds.
- **Modern Riverpod code-gen** — `@riverpod`, generic `Ref` (Riverpod 3), no legacy providers, no hand-edited `*.g.dart`.
- **Idiomatic Dart** — `flutter_lints` / `very_good_analysis` naming, `const` correctness, granular widget extraction.

## Install

```bash
# 1) add this repo as a marketplace
claude plugin marketplace add abdelrahman-abied/mvn-architecture-skill

# 2) install the plugin
claude plugin install mvn-architect@mvn-architecture
```

Or interactively inside Claude Code: `/plugin marketplace add abdelrahman-abied/mvn-architecture-skill` then `/plugin install mvn-architect@mvn-architecture`.

## Use

Once installed, invoke the skill explicitly:

```
/mvn-architect:mvn  scaffold a "profile" feature
```

…or just describe a Flutter/Riverpod task — Claude auto-invokes the skill based on its description (e.g. "add a checkout feature", "refactor this notifier", "this view has business logic in it").

## Repo layout

```
mvn-architecture/
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog
└── mvn-architect/                # the plugin
    ├── .claude-plugin/
    │   └── plugin.json
    └── skills/
        └── mvn/
            └── SKILL.md          # the skill
```

## Versioning

`plugin.json` and the marketplace entry pin `version`. **Bump it on every release** or installed clients won't pull updates (Claude Code treats a fixed `version` as intentional pinning).

## License

Apache-2.0 — see [LICENSE](LICENSE).
