---
name: mvn
description: Scaffold, analyze, and refactor Flutter apps using Clean Architecture + the MVN (Model-View-Notifier) pattern with Riverpod codegen. Use when creating or restructuring a Flutter feature, writing notifiers/states/repositories/usecases/sources, or enforcing the domain/data/presentation layering produced by the "Clean Architecture and MVN with RiverPod" VSCode extension.
---

# Clean Architecture + MVN Engineering Spec

You are a principal Flutter systems architect. Enforce **Clean Architecture** layering together with the **MVN (Model-View-Notifier)** presentation pattern, idiomatic Dart, and modern Riverpod code generation.

MVN was introduced by Abdulrahman Abied: the bridge between View and Model is a Riverpod **`Notifier`** — never a "ViewModel", "Controller", or "VM". This skill is the code-writing companion to the *Clean Architecture and MVN with RiverPod* VSCode extension and must generate code that drops directly into the folders that extension creates.

---

## 📐 1. Canonical Folder Structure (binding spec)

Generate every feature under `lib/features/<feature>/` using **exactly** this tree. Folder names are **singular** (`notifier`, `state`, `view`, `widget`) — match them verbatim.

```
lib/
├── core/                         # cross-feature: theme, network client, errors, DI, utils
└── features/
    └── <feature>/
        ├── domain/               # pure Dart — no Flutter, no Riverpod-of-UI, no data imports
        │   ├── entities/
        │   │   └── <feature>_entity.dart       # business object
        │   ├── repositories/
        │   │   └── <feature>_repository.dart    # ABSTRACT contract (interface)
        │   └── usecases/
        │       └── <feature>_usecase.dart       # one intent per use case
        ├── data/                 # implements domain contracts
        │   ├── models/
        │   │   ├── <feature>_request.dart        # outbound DTO
        │   │   └── <feature>_response.dart        # inbound DTO + toEntity()
        │   ├── repositories/
        │   │   └── <feature>_repository_impl.dart # implements domain repository
        │   └── sources/
        │       └── <feature>_source.dart          # remote/local data source
        └── presentation/         # MVN lives here
            ├── notifier/
            │   └── <feature>_notifier.dart        # extends _$<Feature>Notifier
            ├── state/
            │   └── <feature>_state.dart           # sealed state
            ├── view/
            │   └── <feature>_view.dart            # ConsumerWidget
            └── widget/
                └── <feature>_widget.dart          # granular sub-widgets
```

**Banned:** any `controller`/`controllers` folder or class, any `viewmodel`/`vm` naming. The presentation bridge folder is `notifier/` (singular) and classes end in `Notifier`. *Animation* controllers are the only allowed use of the word "Controller".

---

## 🧭 2. Layer Boundaries & The Dependency Rule

Dependencies point **inward**: `presentation ➔ domain ⬅ data`. Domain is the center and depends on nothing else. Data implements domain. Presentation consumes domain. Runtime data flow is unidirectional: **View ➔ Notifier ➔ UseCase ➔ Repository ➔ Source**.

### A. Domain Layer (`domain/`) — the pure core
* **Entities:** plain, deeply-immutable business objects (`@immutable`, all fields `final`). No JSON, no framework types.
* **Repository contracts:** declare the abstraction with `abstract interface class <Feature>Repository`. The domain owns the contract; it never knows the implementation.
* **Use cases:** one class per intent (e.g. `FetchTrending<Feature>`), constructor-injected with the **abstract** repository, exposing a single `call()`. Keep the use-case *class* free of any `data/` import.
* **No UI Riverpod:** domain code must never `watch`/`read` UI state or notifiers.

### B. Data Layer (`data/`) — the implementation
* **Models are DTOs, not entities.** `<feature>_request.dart` serializes outward (`toJson`); `<feature>_response.dart` parses inward (`fromJson`) and exposes `toEntity()`. Entities never leak JSON; DTOs never leave the data layer.
* **Repository impl:** `<Feature>RepositoryImpl implements <Feature>Repository`, depends on a source, maps responses ➔ entities.
* **Sources:** isolate the transport (REST/db/cache) behind a small interface. A source is a leaf node — it must never read or watch UI state or notifiers.

### C. Composition root (the `@riverpod` provider seam)
The generated providers are the **only** place a concrete implementation is bound to an abstraction. Declare the repository provider in the data layer returning the **abstract** type, so consumers depend on the contract:

```dart
@riverpod
<Feature>Repository <feature>Repository(Ref ref) =>
    <Feature>RepositoryImpl(ref.watch(<feature>SourceProvider));
```

The use-case provider (co-located with the use case under `domain/usecases/`) wires the use case to that repository provider. This provider file is the single sanctioned spot where a domain-adjacent file imports the data provider — the DI seam — and is acceptable precisely because the use-case *class* stays pure.

### D. Presentation Layer (`presentation/`) — MVN
* **Notifier (`notifier/`):** the class name ends in `Notifier` (e.g. `MovieNotifier`) and extends the generated `_$<Feature>Notifier` via `@riverpod`. It orchestrates use cases and maps results onto state. No widgets, no `BuildContext`, no raw repository/source access.
* **State (`state/`):** a `sealed class` with `final` variant subclasses (`Initial`, `Loading`, `Success`, `Error`) enabling compile-time exhaustive `switch`. Deeply immutable.
* **View (`view/`):** a `ConsumerWidget`/`ConsumerStatefulWidget` with **zero** business logic, repository references, or data formatting. Renders state via an exhaustive `switch` expression.
* **Widget (`widget/`):** granular, often `const`, sub-widgets extracted from the view to bound rebuilds.
* **Inter-notifier deps:** when a notifier depends on another's state, establish a reactive link with `ref.watch(otherProvider)` **inside `build()`** — never imperative cross-controller mutation.
* **One-time side effects:** toasts, navigation, sheets ➔ `ref.listen()` inside `build()`. Never store transient events as mutable booleans in the state class.

---

## ⚙️ 3. Riverpod Code-Gen Conventions

* **Modern syntax only:** define providers and notifiers with `@riverpod` + `part '<file>.g.dart';`. Notifiers extend `_$<Name>`. No legacy `StateNotifierProvider`/`ChangeNotifier`.
* **Riverpod 3 `Ref`:** function providers take the generic `Ref ref` — **not** the removed per-provider `*Ref` types (`MovieRepositoryRef` etc. no longer exist).
* **Reads vs watches:** `ref.watch` for reactive dependencies; `ref.read` for one-shot calls inside notifier methods and callbacks.
* **Build step:** generated code via `dart run build_runner build -d` (or `watch`). Never hand-edit `*.g.dart`.

---

## 🎯 4. Dart Conventions, Lints & Performance

Mirror `flutter_lints` + `very_good_analysis`, and enable `riverpod_lint`/`custom_lint`.

* **Naming:** `snake_case` files/folders (`movie_notifier.dart`); `UpperCamelCase` types/enums; `lowerCamelCase` members/vars.
* **`const` everywhere allowed:** const constructors and literals to maximize element caching and cut rebuilds.
* **Exhaustive patterns:** prefer `switch` **expressions** over statements; rely on sealed-class exhaustiveness — no `default` clause that hides a missing state.
* **Targeted rebuilds:** use `ref.watch(provider.select((s) => s.field))` when a widget needs one property of a larger state.
* **Granular extraction:** reject monolithic `build()` methods. If a subtree has its own logic or watches a provider, extract it into its own `ConsumerWidget`/`StatelessWidget` under `widget/`.
* **Direct callbacks:** pass tear-offs when the signature matches — `onPressed: ref.read(p.notifier).load` rather than `onPressed: () => ref.read(p.notifier).load()`.
* **Value equality:** for state/entities that feed `.select()` or comparisons, implement real equality (`freezed` or `equatable`, or manual `==`/`hashCode`) — `@immutable` alone gives only reference equality.

---

## 🛠️ 5. Golden Reference Template (`movie` feature)

```dart
// lib/features/movie/domain/entities/movie_entity.dart
import 'package:meta/meta.dart';

@immutable
class MovieEntity {
  final String id;
  final String title;
  const MovieEntity({required this.id, required this.title});
}
```

```dart
// lib/features/movie/domain/repositories/movie_repository.dart
import '../entities/movie_entity.dart';

abstract interface class MovieRepository {
  Future<List<MovieEntity>> fetchTrending();
}
```

```dart
// lib/features/movie/domain/usecases/movie_usecase.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../entities/movie_entity.dart';
import '../repositories/movie_repository.dart';
// DI seam: the use-case PROVIDER may reference the data provider; the class stays pure.
import '../../data/repositories/movie_repository_impl.dart';

part 'movie_usecase.g.dart';

class FetchTrendingMovies {
  final MovieRepository _repository;
  const FetchTrendingMovies(this._repository);

  Future<List<MovieEntity>> call() => _repository.fetchTrending();
}

@riverpod
FetchTrendingMovies fetchTrendingMovies(Ref ref) =>
    FetchTrendingMovies(ref.watch(movieRepositoryProvider));
```

```dart
// lib/features/movie/data/models/movie_request.dart
import 'package:meta/meta.dart';

@immutable
class MovieRequest {
  final String query;
  const MovieRequest({required this.query});

  Map<String, dynamic> toJson() => {'query': query};
}
```

```dart
// lib/features/movie/data/models/movie_response.dart
import 'package:meta/meta.dart';

import '../../domain/entities/movie_entity.dart';

@immutable
class MovieResponse {
  final String id;
  final String title;
  const MovieResponse({required this.id, required this.title});

  factory MovieResponse.fromJson(Map<String, dynamic> json) => MovieResponse(
        id: json['id'] as String,
        title: json['title'] as String,
      );

  MovieEntity toEntity() => MovieEntity(id: id, title: title);
}
```

```dart
// lib/features/movie/data/sources/movie_source.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../models/movie_response.dart';

part 'movie_source.g.dart';

abstract interface class MovieSource {
  Future<List<MovieResponse>> getTrending();
}

class MovieRemoteSource implements MovieSource {
  @override
  Future<List<MovieResponse>> getTrending() async {
    await Future<void>.delayed(const Duration(milliseconds: 500));
    return const [MovieResponse(id: '42', title: 'Interstellar')];
  }
}

@riverpod
MovieSource movieSource(Ref ref) => MovieRemoteSource();
```

```dart
// lib/features/movie/data/repositories/movie_repository_impl.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../domain/entities/movie_entity.dart';
import '../../domain/repositories/movie_repository.dart';
import '../sources/movie_source.dart';

part 'movie_repository_impl.g.dart';

class MovieRepositoryImpl implements MovieRepository {
  final MovieSource _source;
  const MovieRepositoryImpl(this._source);

  @override
  Future<List<MovieEntity>> fetchTrending() async {
    final responses = await _source.getTrending();
    return responses.map((r) => r.toEntity()).toList();
  }
}

@riverpod
MovieRepository movieRepository(Ref ref) =>
    MovieRepositoryImpl(ref.watch(movieSourceProvider));
```

```dart
// lib/features/movie/presentation/state/movie_state.dart
import 'package:meta/meta.dart';

import '../../domain/entities/movie_entity.dart';

@immutable
sealed class MovieState {
  const MovieState();
}

final class MovieInitial extends MovieState {
  const MovieInitial();
}

final class MovieLoading extends MovieState {
  const MovieLoading();
}

final class MovieSuccess extends MovieState {
  final List<MovieEntity> movies;
  const MovieSuccess(this.movies);
}

final class MovieError extends MovieState {
  final String message;
  const MovieError(this.message);
}
```

```dart
// lib/features/movie/presentation/notifier/movie_notifier.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../domain/usecases/movie_usecase.dart';
import '../state/movie_state.dart';

part 'movie_notifier.g.dart';

@riverpod
class MovieNotifier extends _$MovieNotifier {
  @override
  MovieState build() => const MovieInitial();

  Future<void> loadTrending() async {
    state = const MovieLoading();
    try {
      final movies = await ref.read(fetchTrendingMoviesProvider)();
      state = MovieSuccess(movies);
    } catch (e) {
      state = MovieError(e.toString());
    }
  }
}
```

```dart
// lib/features/movie/presentation/widget/movie_list.dart
import 'package:flutter/material.dart';

import '../../domain/entities/movie_entity.dart';

class MovieList extends StatelessWidget {
  final List<MovieEntity> movies;
  const MovieList({required this.movies, super.key});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: movies.length,
      itemBuilder: (context, index) => ListTile(
        title: Text(movies[index].title),
      ),
    );
  }
}
```

```dart
// lib/features/movie/presentation/view/movie_view.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../notifier/movie_notifier.dart';
import '../state/movie_state.dart';
import '../widget/movie_list.dart';

class MovieView extends ConsumerWidget {
  const MovieView({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // One-time side effects, outside the paint pipeline.
    ref.listen<MovieState>(movieNotifierProvider, (previous, current) {
      if (current is MovieError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(current.message)),
        );
      }
    });

    final state = ref.watch(movieNotifierProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Clean Architecture + MVN')),
      body: switch (state) {
        MovieInitial() => Center(
            child: ElevatedButton(
              onPressed: ref.read(movieNotifierProvider.notifier).loadTrending,
              child: const Text('Fetch Trends'),
            ),
          ),
        MovieLoading() => const Center(child: CircularProgressIndicator()),
        MovieError(message: final m) => Center(child: Text(m)),
        MovieSuccess(movies: final list) => MovieList(movies: list),
      },
    );
  }
}
```

> The extension seeds `presentation/widget/<feature>_widget.dart`; rename it / add descriptively-named widgets (e.g. `movie_list.dart`) in this folder as the UI grows.

---

## 📦 6. Dependencies & Commands

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.6.0   # use ^3.x for Riverpod 3 (generic Ref)
  riverpod_annotation: ^2.6.0

dev_dependencies:
  build_runner: ^2.4.0
  riverpod_generator: ^2.6.0
  custom_lint: ^0.7.0
  riverpod_lint: ^2.6.0
```

```bash
# generate once
dart run build_runner build --delete-conflicting-outputs
# or watch during development
dart run build_runner watch -d
```

When refactoring an existing feature: map each file to its canonical home above, hoist abstract repository contracts into `domain/repositories/`, push implementations to `data/`, split entities (domain) from DTOs (data/models), and rename any `controller`/`viewmodel` to a `Notifier` under `presentation/notifier/`.
