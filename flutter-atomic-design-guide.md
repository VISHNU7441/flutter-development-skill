# Flutter Production Architecture Guide
## Atomic Design + Clean Architecture + Riverpod + GoRouter

> **Agent Instructions:** This document is a strict, executable specification. Every section marked with `[RULE]` is non-negotiable. Sections marked `[PATTERN]` provide the canonical code template to copy. Do not deviate from folder paths, naming conventions, or layer boundaries defined here.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Atomic Design System](#2-atomic-design-system)
3. [Project Structure](#3-project-structure)
4. [Layer Specifications](#4-layer-specifications)
5. [Tech Stack Rules](#5-tech-stack-rules)
6. [Data Flow](#6-data-flow)
7. [Code Patterns](#7-code-patterns)
8. [Naming Conventions](#8-naming-conventions)
9. [Testing Strategy](#9-testing-strategy)
10. [Agent Checklist](#10-agent-checklist)

---

## 1. Architecture Overview

This project uses a **hybrid architecture** combining:

- **Atomic Design** for UI composition (shared + feature-level)
- **Clean Architecture** for feature internals (Data → Domain → Presentation)
- **Hexagonal Architecture** for swappable data sources

```
┌─────────────────────────────────────────┐
│            Presentation Layer           │
│   (Atomic Design: Pages → Organisms)    │
├─────────────────────────────────────────┤
│              Domain Layer               │
│   (UseCases, Entities, Repo Interfaces) │
├─────────────────────────────────────────┤
│               Data Layer                │
│   (Repositories, Sources, DTOs/Models)  │
└─────────────────────────────────────────┘
```

**[RULE]** The dependency rule flows strictly **inward**: Presentation → Domain ← Data. The Domain layer must never import from Presentation or Data layers.

---

## 2. Atomic Design System

### Hierarchy

```
Pages
  └── Templates
        └── Organisms
              └── Molecules
                    └── Atoms
```

### 2.1 Atoms

**Definition:** The smallest, indivisible UI unit. Stateless. No business logic. No state management integration.

**[RULE]** Atoms must:
- Accept only primitive types or simple data classes via constructor
- Never call any Provider, Bloc, or Notifier
- Never import from `features/` or `domain/`
- Be placed in `lib/shared/atoms/`

**[RULE]** Atoms must NOT:
- Contain `if` logic based on app state
- Make network calls
- Use `BuildContext` to read providers

**Examples:** `AppButton`, `AppText`, `AppTextField`, `AppAvatar`, `AppDivider`, `AppBadge`

**[PATTERN] Atom Template:**
```dart
// lib/shared/atoms/app_button.dart
import 'package:flutter/material.dart';

class AppButton extends StatelessWidget {
  const AppButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.isLoading = false,
    this.variant = AppButtonVariant.primary,
  });

  final String label;
  final VoidCallback? onPressed;
  final bool isLoading;
  final AppButtonVariant variant;

  @override
  Widget build(BuildContext context) {
    // Pure widget logic only — no Provider reads
    return ElevatedButton(
      onPressed: isLoading ? null : onPressed,
      child: isLoading
          ? const CircularProgressIndicator.adaptive()
          : Text(label),
    );
  }
}

enum AppButtonVariant { primary, secondary, destructive }
```

---

### 2.2 Molecules

**Definition:** A composition of 2+ Atoms that performs one focused UI task (Single Responsibility Principle).

**[RULE]** Molecules must:
- Be composed only of Atoms (or other Molecules)
- Perform exactly one UI responsibility
- Be placed in `lib/shared/molecules/`
- Expose all data via constructor parameters (no internal state fetching)

**[RULE]** Molecules must NOT:
- Import from `features/`
- Contain business logic
- Directly call APIs or repositories

**Examples:** `SearchBar` (AppTextField + AppButton), `UserTile` (AppAvatar + AppText), `QuantitySelector` (AppButton + AppText + AppButton)

**[PATTERN] Molecule Template:**
```dart
// lib/shared/molecules/user_tile.dart
import 'package:flutter/material.dart';
import '../atoms/app_avatar.dart';
import '../atoms/app_text.dart';

class UserTile extends StatelessWidget {
  const UserTile({
    super.key,
    required this.name,
    required this.avatarUrl,
    this.subtitle,
    this.onTap,
  });

  final String name;
  final String avatarUrl;
  final String? subtitle;
  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: AppAvatar(imageUrl: avatarUrl),
      title: AppText(text: name),
      subtitle: subtitle != null ? AppText(text: subtitle!, style: AppTextStyle.caption) : null,
      onTap: onTap,
    );
  }
}
```

---

### 2.3 Organisms

**Definition:** A complex, self-contained UI section. The first level where state management integration is permitted.

**[RULE]** Organisms must:
- Be composed of Molecules and/or Atoms
- Be placed in `lib/shared/organisms/` (global) or `lib/features/<feature>/presentation/components/` (feature-specific)
- Represent a meaningful section of the UI (e.g., a navigation bar, a product list, a comment thread)

**[RULE]** Organisms CAN:
- Use `ConsumerWidget` (Riverpod) or `BlocBuilder` to read a specific slice of state
- Contain local UI state via `StatefulWidget` or `useState`

**[RULE]** Organisms must NOT:
- Trigger navigation directly (pass callbacks up to Pages)
- Execute use cases directly (delegate to Providers/Notifiers)

**Examples:** `CustomAppBar`, `AppDrawer`, `ProductCardList`, `NotificationFeed`

---

### 2.4 Templates

**Definition:** The layout skeleton of a screen. Defines spatial positioning without real data.

**[RULE]** Templates must:
- Be placed in `lib/features/<feature>/presentation/` as `<feature>_template.dart`
- Use `Scaffold`, `Column`, `Row`, `Stack`, `Slivers` to define structure
- Accept Organisms and Molecules as constructor parameters (children), not data directly
- Handle responsive layout (different layouts per screen size)

**[RULE]** Templates must NOT:
- Fetch data
- Read from any Provider or Bloc
- Contain business logic

**[PATTERN] Template:**
```dart
// lib/features/product_list/presentation/product_list_template.dart
class ProductListTemplate extends StatelessWidget {
  const ProductListTemplate({
    super.key,
    required this.appBar,
    required this.productList,
    required this.filterBar,
    this.floatingActionButton,
  });

  final Widget appBar;
  final Widget productList;
  final Widget filterBar;
  final Widget? floatingActionButton;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: PreferredSize(
        preferredSize: const Size.fromHeight(56),
        child: appBar,
      ),
      body: Column(
        children: [filterBar, Expanded(child: productList)],
      ),
      floatingActionButton: floatingActionButton,
    );
  }
}
```

---

### 2.5 Pages (Screens)

**Definition:** The top-level entry point for a route. Injects real data into Templates.

**[RULE]** Pages must:
- Be placed in `lib/features/<feature>/presentation/pages/`
- Be the GoRouter destination (`/`)
- Read from Providers/Notifiers and pass data down to the Template
- Handle loading, error, and empty states

**[RULE]** Pages must NOT:
- Contain direct widget layout (delegate to Templates)
- Make `http` calls directly
- Import from other features' `presentation/` layer

**[PATTERN] Page:**
```dart
// lib/features/product_list/presentation/pages/product_list_page.dart
class ProductListPage extends ConsumerWidget {
  const ProductListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(productListProvider);

    return state.when(
      loading: () => const ProductListTemplate(
        appBar: CustomAppBar(title: 'Products'),
        filterBar: ShimmerFilterBar(),
        productList: ProductListShimmer(),
      ),
      error: (err, _) => ErrorPage(message: err.toString()),
      data: (products) => ProductListTemplate(
        appBar: CustomAppBar(title: 'Products'),
        filterBar: FilterBarOrganism(onFilterChanged: (f) => ref.read(productListProvider.notifier).applyFilter(f)),
        productList: ProductCardList(products: products),
      ),
    );
  }
}
```

---

## 3. Project Structure

**[RULE]** This is the canonical folder structure. Do not rename, relocate, or merge any of these directories.

```
lib/
├── core/                               # App-wide singletons — initialized once at startup
│   ├── network/
│   │   ├── dio_client.dart             # Dio singleton with base options
│   │   └── interceptors/
│   │       ├── auth_interceptor.dart
│   │       └── logging_interceptor.dart
│   ├── local_db/
│   │   └── database_service.dart       # Sqflite initialization
│   ├── router/
│   │   └── app_router.dart             # GoRouter configuration
│   └── theme/
│       ├── app_colors.dart             # Color tokens
│       ├── app_typography.dart         # TextStyle definitions
│       └── app_theme.dart              # ThemeData assembly
│
├── shared/                             # Global, feature-agnostic UI Kit
│   ├── atoms/
│   ├── molecules/
│   └── organisms/
│
└── features/                           # One folder per product feature
    └── <feature_name>/                 # e.g., product_list, auth, profile
        ├── data/
        │   ├── sources/
        │   │   ├── remote/
        │   │   │   └── <feature>_remote_source.dart
        │   │   └── local/
        │   │       └── <feature>_local_source.dart
        │   ├── models/
        │   │   └── <feature>_model.dart   # DTO with fromJson / toJson
        │   └── repositories/
        │       └── <feature>_repository_impl.dart
        │
        ├── domain/
        │   ├── entities/
        │   │   └── <feature>_entity.dart  # Freezed, immutable
        │   ├── repositories/
        │   │   └── <feature>_repository.dart  # abstract interface
        │   └── usecases/
        │       └── get_<feature>_usecase.dart
        │
        └── presentation/
            ├── providers/
            │   └── <feature>_provider.dart
            ├── components/              # Feature-scoped Atoms/Molecules/Organisms
            └── pages/
                └── <feature>_page.dart
```

---

## 4. Layer Specifications

### 4.1 Data Layer

**Purpose:** Adapts external data (API, local DB) into domain entities.

**Components:**

| Component | Responsibility | Allowed Imports |
|---|---|---|
| Remote Source | Dio/Chopper API calls | `dio`, `retrofit` |
| Local Source | Sqflite read/write | `sqflite` |
| Model (DTO) | JSON serialization | `json_serializable`, `freezed` |
| Repository Impl | Calls sources, maps Model → Entity | Domain interfaces only |

**[RULE]** Models are DTOs only. They must implement `fromJson` and `toJson`. They must NOT be used beyond the Data layer boundary — convert to Entities before returning.

**[PATTERN] Model:**
```dart
// lib/features/product_list/data/models/product_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import '../../domain/entities/product_entity.dart';

part 'product_model.freezed.dart';
part 'product_model.g.dart';

@freezed
class ProductModel with _$ProductModel {
  const factory ProductModel({
    required String id,
    required String name,
    required double price,
    @JsonKey(name: 'image_url') required String imageUrl,
  }) = _ProductModel;

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);
}

extension ProductModelMapper on ProductModel {
  ProductEntity toEntity() => ProductEntity(
    id: id,
    name: name,
    price: price,
    imageUrl: imageUrl,
  );
}
```

**[PATTERN] Repository Implementation:**
```dart
// lib/features/product_list/data/repositories/product_repository_impl.dart
class ProductRepositoryImpl implements ProductRepository {
  const ProductRepositoryImpl({
    required this.remoteSource,
    required this.localSource,
  });

  final ProductRemoteSource remoteSource;
  final ProductLocalSource localSource;

  @override
  Future<Either<Failure, List<ProductEntity>>> getProducts() async {
    try {
      final models = await remoteSource.fetchProducts();
      final entities = models.map((m) => m.toEntity()).toList();
      await localSource.cacheProducts(models); // cache for offline
      return Right(entities);
    } on DioException catch (e) {
      // Fallback to local cache on network error
      final cached = await localSource.getCachedProducts();
      if (cached.isNotEmpty) return Right(cached.map((m) => m.toEntity()).toList());
      return Left(NetworkFailure(message: e.message ?? 'Network error'));
    }
  }
}
```

---

### 4.2 Domain Layer

**Purpose:** Pure business logic. Zero framework dependencies.

**[RULE]** The Domain layer must:
- Contain only pure Dart (no Flutter, no Dio, no Riverpod imports)
- Define repository contracts as abstract classes
- Express business rules in UseCase classes

**[RULE]** Entities must use `Freezed` for immutability.

**[PATTERN] Entity:**
```dart
// lib/features/product_list/domain/entities/product_entity.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product_entity.freezed.dart';

@freezed
class ProductEntity with _$ProductEntity {
  const factory ProductEntity({
    required String id,
    required String name,
    required double price,
    required String imageUrl,
  }) = _ProductEntity;
}
```

**[PATTERN] Repository Interface:**
```dart
// lib/features/product_list/domain/repositories/product_repository.dart
import 'package:dartz/dartz.dart';
import '../entities/product_entity.dart';
import '../../../../core/error/failure.dart';

abstract interface class ProductRepository {
  Future<Either<Failure, List<ProductEntity>>> getProducts();
  Future<Either<Failure, ProductEntity>> getProductById(String id);
}
```

**[PATTERN] UseCase:**
```dart
// lib/features/product_list/domain/usecases/get_products_usecase.dart
class GetProductsUseCase {
  const GetProductsUseCase(this._repository);

  final ProductRepository _repository;

  Future<Either<Failure, List<ProductEntity>>> call() {
    return _repository.getProducts();
  }
}
```

---

### 4.3 Presentation Layer

**Purpose:** UI rendering and state management via Riverpod.

**[RULE]** Providers must:
- Use `AsyncNotifier` for async operations
- Use `Notifier` for synchronous state
- Call UseCases — never call repositories directly
- Be placed in `lib/features/<feature>/presentation/providers/`

**[PATTERN] Provider:**
```dart
// lib/features/product_list/presentation/providers/product_list_provider.dart
@riverpod
class ProductList extends _$ProductList {
  @override
  Future<List<ProductEntity>> build() async {
    final useCase = ref.read(getProductsUseCaseProvider);
    final result = await useCase();
    return result.fold(
      (failure) => throw failure,
      (products) => products,
    );
  }

  Future<void> refresh() => ref.refresh(productListProvider.future);
}
```

---

## 5. Tech Stack Rules

### 5.1 Networking — Dio

**[RULE]** One Dio instance, created in `core/network/dio_client.dart` and provided via Riverpod.

```dart
@riverpod
Dio dio(DioRef ref) {
  return Dio(BaseOptions(
    baseUrl: Env.apiBaseUrl,
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 30),
    headers: {'Accept': 'application/json'},
  ))
    ..interceptors.addAll([
      AuthInterceptor(ref),
      LoggingInterceptor(),
    ]);
}
```

### 5.2 Local Storage — Sqflite

**[RULE]** All Sqflite access must go through `DatabaseService`. No feature may open a raw database connection.

### 5.3 State Management — Riverpod

**[RULE]** Use `@riverpod` code generation (riverpod_generator). Never use legacy `Provider()` constructors.

**[RULE]** Provider scoping:
- Global providers → `lib/core/` or `lib/shared/`
- Feature-scoped providers → `lib/features/<feature>/presentation/providers/`

### 5.4 Routing — GoRouter

**[RULE]** All routes are defined in `lib/core/router/app_router.dart`. No feature may define its own isolated GoRouter.

**[RULE]** Route paths use `kebab-case`. Route names use `camelCase` constants.

```dart
// lib/core/router/app_router.dart
@riverpod
GoRouter appRouter(AppRouterRef ref) {
  return GoRouter(
    initialLocation: '/products',
    routes: [
      GoRoute(
        path: '/products',
        name: AppRoutes.productList,
        builder: (_, __) => const ProductListPage(),
        routes: [
          GoRoute(
            path: ':id',
            name: AppRoutes.productDetail,
            builder: (_, state) => ProductDetailPage(
              productId: state.pathParameters['id']!,
            ),
          ),
        ],
      ),
    ],
  );
}

abstract class AppRoutes {
  static const productList = 'productList';
  static const productDetail = 'productDetail';
}
```

### 5.5 Code Generation

**[RULE]** Run `dart run build_runner build --delete-conflicting-outputs` after any change to:
- Freezed entities or models
- Riverpod providers with `@riverpod`
- JsonSerializable models

---

## 6. Data Flow

The complete lifecycle of a user action:

```
1. USER ACTION
   └─► Atom widget (e.g., AppButton.onPressed)

2. EVENT DISPATCH
   └─► Organism or Page calls ref.read(provider.notifier).someMethod()

3. NOTIFIER (Presentation)
   └─► Calls UseCase: final result = await _getProductsUseCase()

4. USE CASE (Domain)
   └─► Calls Repository Interface: _repository.getProducts()

5. REPOSITORY IMPL (Data)
   └─► Decides: remoteSource or localSource?
   └─► Calls Source (Dio / Sqflite)

6. RETURN PATH
   Source → Model → (map) → Entity → UseCase → Notifier → AsyncValue

7. UI RE-RENDER
   └─► Page watches AsyncValue, passes data to Template → Organisms → Atoms
```

**[RULE]** Navigation is always triggered from the **Page** level via `context.go()` or `context.push()`. Organisms must never call GoRouter directly — they receive `VoidCallback` or `void Function(T)` from the Page.

---

## 7. Code Patterns

### 7.1 Error Handling

**[RULE]** Use a sealed `Failure` class hierarchy. Never throw raw exceptions across layer boundaries.

```dart
// lib/core/error/failure.dart
sealed class Failure {
  const Failure({required this.message});
  final String message;
}

final class NetworkFailure extends Failure {
  const NetworkFailure({required super.message});
}

final class CacheFailure extends Failure {
  const CacheFailure({required super.message});
}

final class ServerFailure extends Failure {
  const ServerFailure({required super.message, this.statusCode});
  final int? statusCode;
}
```

### 7.2 Dependency Injection

**[RULE]** All dependencies are injected via Riverpod providers. No service locator (GetIt) unless explicitly approved.

```dart
// Wire UseCases to their repositories via providers
@riverpod
GetProductsUseCase getProductsUseCase(GetProductsUseCaseRef ref) {
  return GetProductsUseCase(ref.read(productRepositoryProvider));
}

@riverpod
ProductRepository productRepository(ProductRepositoryRef ref) {
  return ProductRepositoryImpl(
    remoteSource: ref.read(productRemoteSourceProvider),
    localSource: ref.read(productLocalSourceProvider),
  );
}
```

### 7.3 Responsiveness in Templates

**[RULE]** Templates handle screen-size breakpoints. Atoms and Molecules are size-agnostic.

```dart
class ProductListTemplate extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.sizeOf(context).width;
    final crossAxisCount = switch (width) {
      < 600 => 2,   // Mobile
      < 1024 => 3,  // Tablet
      _ => 4,        // Desktop
    };
    return GridView.builder(gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: crossAxisCount), ...);
  }
}
```

---

## 8. Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Files | `snake_case.dart` | `product_list_page.dart` |
| Classes | `PascalCase` | `ProductListPage` |
| Providers | `camelCase` + suffix `Provider` | `productListProvider` |
| Notifiers | `PascalCase` + no suffix | `ProductList` (generates `productListProvider`) |
| Entities | `PascalCase` + `Entity` | `ProductEntity` |
| Models (DTOs) | `PascalCase` + `Model` | `ProductModel` |
| UseCases | `PascalCase` + `UseCase` | `GetProductsUseCase` |
| Repository interface | `PascalCase` + `Repository` | `ProductRepository` |
| Repository impl | `PascalCase` + `RepositoryImpl` | `ProductRepositoryImpl` |
| Remote source | `PascalCase` + `RemoteSource` | `ProductRemoteSource` |
| Local source | `PascalCase` + `LocalSource` | `ProductLocalSource` |
| Route paths | `kebab-case` | `/product-detail` |
| Route name constants | `camelCase` | `AppRoutes.productDetail` |
| Atoms | `App` prefix (if shared) | `AppButton`, `AppText` |
| Feature organisms | Descriptive noun | `ProductCardList` |

---

## 9. Testing Strategy

### Test Placement

```
test/
├── shared/
│   ├── atoms/          # Widget tests for all Atoms
│   └── molecules/      # Widget tests for all Molecules
└── features/
    └── <feature>/
        ├── data/        # Unit tests for models, sources, repo impl
        ├── domain/      # Unit tests for use cases (mock repositories)
        └── presentation/ # Widget tests for Pages; provider tests with overrides
```

### Testing Rules

**[RULE]** Atoms and Molecules must have widget tests with 100% coverage.

**[RULE]** UseCases must have unit tests with mocked repository interfaces.

**[RULE]** Providers must be tested using `ProviderContainer` with repository overrides.

**[PATTERN] UseCase Test:**
```dart
void main() {
  late MockProductRepository mockRepo;
  late GetProductsUseCase useCase;

  setUp(() {
    mockRepo = MockProductRepository();
    useCase = GetProductsUseCase(mockRepo);
  });

  test('returns product list on success', () async {
    final products = [ProductEntity(id: '1', name: 'Widget', price: 9.99, imageUrl: '')];
    when(() => mockRepo.getProducts()).thenAnswer((_) async => Right(products));

    final result = await useCase();

    expect(result, Right(products));
    verify(() => mockRepo.getProducts()).called(1);
  });
}
```

---

## 10. Agent Checklist

When adding a new feature, an AI agent must complete every item in this checklist in order:

### Step 1: Domain Layer (Pure Dart — no Flutter)
- [ ] Create `lib/features/<feature>/domain/entities/<feature>_entity.dart` using `@freezed`
- [ ] Create `lib/features/<feature>/domain/repositories/<feature>_repository.dart` as `abstract interface class`
- [ ] Create `lib/features/<feature>/domain/usecases/get_<feature>_usecase.dart`
- [ ] Run `build_runner` to generate Freezed files

### Step 2: Data Layer
- [ ] Create `lib/features/<feature>/data/models/<feature>_model.dart` with `@freezed` + `@JsonSerializable`
- [ ] Add `toEntity()` extension on the Model
- [ ] Create `lib/features/<feature>/data/sources/remote/<feature>_remote_source.dart`
- [ ] Create `lib/features/<feature>/data/sources/local/<feature>_local_source.dart`
- [ ] Create `lib/features/<feature>/data/repositories/<feature>_repository_impl.dart` implementing the domain interface
- [ ] Run `build_runner` to generate JSON serialization files

### Step 3: Providers (Presentation — Riverpod wiring)
- [ ] Create Riverpod providers for: RemoteSource, LocalSource, RepositoryImpl, UseCase, Notifier
- [ ] Run `build_runner` to generate provider files

### Step 4: UI (Atomic Design)
- [ ] Create shared Atoms/Molecules if new reusable primitives are needed in `lib/shared/`
- [ ] Create feature-specific Organisms in `lib/features/<feature>/presentation/components/`
- [ ] Create Template in `lib/features/<feature>/presentation/<feature>_template.dart`
- [ ] Create Page in `lib/features/<feature>/presentation/pages/<feature>_page.dart`

### Step 5: Routing
- [ ] Register the new Page as a `GoRoute` in `lib/core/router/app_router.dart`
- [ ] Add the route name constant to `AppRoutes`

### Step 6: Testing
- [ ] Write unit tests for all new UseCases
- [ ] Write widget tests for all new Atoms and Molecules
- [ ] Write provider tests with mocked repository

### Step 7: Validation
- [ ] `flutter analyze` returns zero errors
- [ ] `flutter test` passes all tests
- [ ] Confirm no cross-layer import violations (Data layer must not import Presentation; Domain must not import Data or Presentation)

---

## Appendix: Import Boundary Reference

| From ↓ \ To → | `core/` | `shared/` | `data/` | `domain/` | `presentation/` |
|---|---|---|---|---|---|
| `core/` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `shared/` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `data/` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `domain/` | ❌ | ❌ | ❌ | ✅ | ❌ |
| `presentation/` | ✅ | ✅ | ❌ | ✅ | ✅ |

> **[RULE]** Any import that crosses a ❌ boundary is an architecture violation and must be refactored immediately.
