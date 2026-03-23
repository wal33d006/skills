# Refactor Flutter Project to Clean Architecture with Cubit-Driven Navigation

Refactor this Flutter project end-to-end to follow a strict Clean Architecture pattern with Cubit state management and cubit-driven navigation. Follow every step precisely and completely. Do not skip any layer, feature, or file.

If the user names one or more specific features (e.g. `/refactor recording settings`), apply **only** the navigation refactor (Steps 1 and 9–12) to those features and skip Steps 2–8. Otherwise, run the full refactor.

---

## Architecture Overview

```
lib/
├── core/
│   ├── domain/                  # Pure business logic — NO Flutter, NO external SDKs
│   │   ├── models/              # Domain entities
│   │   ├── repositories/        # Abstract interfaces
│   │   ├── use_cases/           # Business rules (orchestrate repos + stores)
│   │   ├── failures/            # Typed failure classes
│   │   └── stores/              # Global app state (long-lived Cubits)
│   └── data/
│       ├── models/              # JSON/DTO classes (fromJson, toJson, toDomain)
│       └── repositories/        # External API/DB implementations of domain interfaces
├── features/
│   └── <feature>/               # 5 files per screen
│       ├── <feature>_initial_params.dart
│       ├── <feature>_state.dart
│       ├── <feature>_cubit.dart
│       ├── <feature>_navigator.dart
│       └── <feature>_page.dart
└── navigation/
    ├── app_navigator.dart
    ├── close_route.dart
    └── error_dialog_route.dart
```

### Strict Dependency Rules (never violate these)

```
Page → Cubit → UseCase → Repository interface → External API/DB
                       ↘ Store (global state side-effects)
```

- **Pages** only call cubit methods. No logic, no direct store/repo access.
- **Cubits** only call use cases and the navigator. Never call repositories or external APIs directly.
- **Use cases** only call repository interfaces and stores. Never call external APIs or data-layer classes.
- **Repository interfaces** (domain) define contracts using `Either<Failure, T>` from `dartz`.
- **Repository implementations** (data) call external APIs, convert JSON ↔ domain models, return `Either`.
- **JSON models** (data) handle serialization only. They never appear above the data layer.
- **Domain models** are used everywhere except inside JSON models.

### Navigation Rule (non-negotiable)

`InitialParams` = **data only**. Cubits own all navigation through their navigator.

```
WRONG: InitialParams has onViewItems, onBack, onSignIn  →  parent wires spaghetti callbacks
RIGHT: InitialParams has data only  →  Cubit calls navigator.openItems(ItemsInitialParams(...))
```

**Navigation callbacks — never put in InitialParams:**
- Opens another screen (`onViewItems`, `onViewProfile`, `onViewDetails`)
- Pops the current screen (`onBack`, `onClose`, `onDismiss`)
- Triggers a screen transition after an action (`onSignIn`, `onSignOut`, `onSaved`)

**Data callbacks — allowed in InitialParams:**
- Returns a result to the caller (`onItemUpdated(Item)`, `onRecordingComplete(Entry)`)

---

## Step 1 — Explore the Project

1. Read `pubspec.yaml` fully — record the **package name** and all existing dependencies.
2. Read `lib/main.dart` fully.
3. List all files under `lib/` recursively.
4. For each existing feature, read all files and note: state managed, actions handled, screens navigated to, data fetched/written, and external services used.
5. Identify the backend (Firebase, REST API, etc.) and the auth mechanism.

Record all findings before writing any code.

Add these packages to `pubspec.yaml` if missing:

```yaml
dependencies:
  flutter_bloc: ^8.0.0
  get_it: ^7.0.0
  equatable: ^2.0.0
  dartz: ^0.10.1
  # keep all existing packages
```

> **PACKAGE_NAME** — replace with the actual package name from `pubspec.yaml` in every file you write.

---

## Step 2 — Domain Models

**Location:** `lib/core/domain/models/`

Domain models are pure Dart classes. They extend `Equatable`, have `copyWith`, and a `factory empty()`.

```dart
// lib/core/domain/models/product.dart
import 'package:equatable/equatable.dart';

class Product extends Equatable {
  final String id;
  final String title;
  final double price;
  final ProductCategory category;

  const Product({
    required this.id,
    required this.title,
    required this.price,
    required this.category,
  });

  factory Product.empty() => const Product(
        id: '',
        title: '',
        price: 0.0,
        category: ProductCategory.unknown,
      );

  Product copyWith({
    String? id,
    String? title,
    double? price,
    ProductCategory? category,
  }) =>
      Product(
        id: id ?? this.id,
        title: title ?? this.title,
        price: price ?? this.price,
        category: category ?? this.category,
      );

  @override
  List<Object?> get props => [id]; // unique identifier only
}

enum ProductCategory { unknown, electronics, clothing }
```

Rules:
- `final` fields only — domain models are immutable
- Always `extends Equatable` — use the unique identifier in `props`
- Always `factory empty()` for safe empty states
- Always `copyWith` with all fields nullable
- No `fromJson`, no external SDK imports — those belong in the data layer
- Enums go in the same file or a nearby file in `domain/models/`

---

## Step 3 — Failures

**Location:** `lib/core/domain/failures/`

### `lib/core/domain/failures/displayable_failure.dart`

```dart
class DisplayableFailure {
  const DisplayableFailure({required this.title, required this.message});

  DisplayableFailure.commonError([String? message])
      : title = 'Error',
        message = message ?? 'Something went wrong, please try again later';

  final String title;
  final String message;
}

abstract class HasDisplayableFailure {
  DisplayableFailure displayableFailure();
}
```

### One failure class per use case / operation

```dart
// lib/core/domain/failures/get_product_failure.dart
import 'package:PACKAGE_NAME/core/domain/failures/displayable_failure.dart';

class GetProductFailure implements HasDisplayableFailure {
  const GetProductFailure.unknown([this.cause]) : type = GetProductFailureType.unknown;
  const GetProductFailure.notFound([this.cause]) : type = GetProductFailureType.notFound;

  final GetProductFailureType type;
  final Object? cause;

  @override
  DisplayableFailure displayableFailure() {
    switch (type) {
      case GetProductFailureType.notFound:
        return const DisplayableFailure(title: 'Not Found', message: 'Product not found.');
      case GetProductFailureType.unknown:
        return DisplayableFailure.commonError();
    }
  }

  @override
  String toString() => 'GetProductFailure{type: $type, cause: $cause}';
}

enum GetProductFailureType { unknown, notFound }
```

Rules:
- Named constructors map to enum variants (`.unknown`, `.notFound`, `.unauthorized`, etc.)
- `displayableFailure()` returns a user-readable message per variant via `switch`
- Always include `cause` to capture the underlying exception for logging
- Never throw exceptions out of repositories — wrap in failure types and return `left(failure)`

---

## Step 4 — Repository Interfaces (Domain)

**Location:** `lib/core/domain/repositories/`

Abstract classes only. Define the contract using `Either<Failure, SuccessType>`. No implementation details.

```dart
// lib/core/domain/repositories/product_repository.dart
import 'package:dartz/dartz.dart';
import 'package:PACKAGE_NAME/core/domain/failures/get_product_failure.dart';
import 'package:PACKAGE_NAME/core/domain/failures/update_product_failure.dart';
import 'package:PACKAGE_NAME/core/domain/models/product.dart';

abstract class ProductRepository {
  Future<Either<GetProductFailure, Product>> getProductById(String id);
  Future<Either<GetProductFailure, List<Product>>> getProducts();
  Future<Either<UpdateProductFailure, Unit>> createProduct(Product product);
  Future<Either<UpdateProductFailure, Unit>> updateProduct(Product product);
  Future<Either<UpdateProductFailure, Unit>> deleteProduct(String id);
}
```

Rules:
- `abstract class` — never `abstract interface`
- Return types always `Future<Either<FailureType, SuccessType>>`
- Use `Unit` from `dartz` for void success
- Only domain models and failure types here — no JSON, no external SDK types
- One repository interface per domain aggregate

---

## Step 5 — JSON Models (Data Layer)

**Location:** `lib/core/data/models/`

JSON models bridge the data source to the domain layer. Three responsibilities: `fromJson`, `toJson`, `toDomain()`.

```dart
// lib/core/data/models/product_json.dart
import 'package:PACKAGE_NAME/core/domain/models/product.dart';

class ProductJson {
  final String id;
  final String title;
  final double price;
  final String category;

  ProductJson({required this.id, required this.title, required this.price, required this.category});

  factory ProductJson.fromJson(Map<String, dynamic> json) => ProductJson(
        id: json['id'] as String? ?? '',
        title: json['title'] as String? ?? '',
        price: (json['price'] as num?)?.toDouble() ?? 0.0,
        category: json['category'] as String? ?? '',
      );

  Map<String, dynamic> toJson() => {
        'id': id,
        'title': title,
        'price': price,
        'category': category,
      };

  // The ONLY place where raw data → domain conversion happens
  Product toDomain() => Product(
        id: id,
        title: title,
        price: price,
        category: _parseCategory(category),
      );

  static ProductCategory _parseCategory(String value) {
    switch (value) {
      case 'electronics': return ProductCategory.electronics;
      case 'clothing': return ProductCategory.clothing;
      default: return ProductCategory.unknown;
    }
  }
}
```

Rules:
- All `fromJson` fields use `as Type?` with `?? defaultValue` — never assume JSON is clean
- `toDomain()` is the ONLY place that creates a domain model from raw data
- Enums serialized as strings — parse in a private static method
- Lists: `(json['field'] as List?)?.cast<String>() ?? []`
- Never import this class above the data layer

**For Firestore Timestamps:**

```dart
// In fromJson:
final ts = json['createdAt'] as Timestamp?;
final createdAt = ts?.toDate() ?? DateTime.now();

// In toJson:
'createdAt': FieldValue.serverTimestamp(),
```

---

## Step 6 — Repository Implementations (Data Layer)

**Location:** `lib/core/data/repositories/`

Concrete classes that `implement` the domain interface. Call external APIs, use JSON models, wrap everything in `Either`.

```dart
// lib/core/data/repositories/firebase_product_repository.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:dartz/dartz.dart';
import 'package:PACKAGE_NAME/core/data/models/product_json.dart';
import 'package:PACKAGE_NAME/core/domain/failures/get_product_failure.dart';
import 'package:PACKAGE_NAME/core/domain/failures/update_product_failure.dart';
import 'package:PACKAGE_NAME/core/domain/models/product.dart';
import 'package:PACKAGE_NAME/core/domain/repositories/product_repository.dart';

class FirebaseProductRepository implements ProductRepository {
  final CollectionReference _collection =
      FirebaseFirestore.instance.collection('products');

  @override
  Future<Either<GetProductFailure, Product>> getProductById(String id) async {
    try {
      final doc = await _collection.doc(id).get();
      if (doc.data() == null) return left(const GetProductFailure.notFound());
      return right(ProductJson.fromJson(doc.data() as Map<String, dynamic>).toDomain());
    } catch (ex) {
      return left(GetProductFailure.unknown(ex));
    }
  }

  @override
  Future<Either<GetProductFailure, List<Product>>> getProducts() async {
    try {
      final snapshot = await _collection.orderBy('createdAt', descending: true).get();
      final products = snapshot.docs
          .map((doc) => ProductJson.fromJson(doc.data() as Map<String, dynamic>).toDomain())
          .toList();
      return right(products);
    } catch (ex) {
      return left(GetProductFailure.unknown(ex));
    }
  }

  @override
  Future<Either<UpdateProductFailure, Unit>> updateProduct(Product product) async {
    try {
      final json = ProductJson(
        id: product.id, title: product.title, price: product.price,
        category: product.category.name,
      ).toJson();
      await _collection.doc(product.id).set(json, SetOptions(merge: true));
      return right(unit);
    } catch (ex) {
      return left(UpdateProductFailure.unknown(ex));
    }
  }

  // ... implement remaining methods following the same pattern
}
```

Rules:
- Always `implements DomainRepository`
- Every method wrapped in `try/catch` returning `left(Failure)` on error
- Firebase → domain: `JsonModel.fromJson(doc.data()).toDomain()`
- Domain → Firebase: construct `JsonModel` from domain fields, call `.toJson()`
- `SetOptions(merge: true)` for updates

---

## Step 7 — Global Stores

**Location:** `lib/core/domain/stores/`

Stores are long-lived Cubits that hold app-wide state (current user, theme, etc.).

```dart
// lib/core/domain/stores/user/user_state.dart
import 'package:PACKAGE_NAME/core/domain/models/user.dart';

class UserState {
  final User user;
  const UserState({required this.user});
}

// lib/core/domain/stores/user/user_store.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:PACKAGE_NAME/core/domain/models/user.dart';
import 'user_state.dart';

class UserStore extends Cubit<UserState> {
  UserStore() : super(UserState(user: User.empty()));

  void setUser(User user) => emit(UserState(user: user));
  void clearUser() => emit(UserState(user: User.empty()));

  User get user => state.user;
  bool get isAuthenticated => !state.user.isEmpty;
}
```

Rules:
- Registered as `registerLazySingleton` in DI — one instance for the whole app lifetime
- Use cases update stores after successful operations (e.g. `UserStore.setUser` after login)
- Cubits can read from stores and listen to their stream for reactive updates:
  ```dart
  void onInit() {
    emit(state.copyWith(user: _userStore.user));
    _userStore.stream.listen((s) => emit(state.copyWith(user: s.user)));
  }
  ```
- Never access stores directly from pages

---

## Step 8 — Use Cases

**Location:** `lib/core/domain/use_cases/`

Use cases orchestrate one business operation. Inject repository **interfaces**, may update stores, return `Either`.

```dart
// lib/core/domain/use_cases/get_product_use_case.dart
import 'package:dartz/dartz.dart';
import 'package:PACKAGE_NAME/core/domain/failures/get_product_failure.dart';
import 'package:PACKAGE_NAME/core/domain/models/product.dart';
import 'package:PACKAGE_NAME/core/domain/repositories/product_repository.dart';

class GetProductUseCase {
  final ProductRepository _productRepository;
  GetProductUseCase(this._productRepository);

  Future<Either<GetProductFailure, Product>> execute(String productId) =>
      _productRepository.getProductById(productId);
}
```

```dart
// Use case that also updates a store after success
class LoginUseCase {
  final AuthRepository _authRepository;
  final UserStore _userStore;
  LoginUseCase(this._authRepository, this._userStore);

  Future<Either<LoginFailure, Unit>> execute(String email, String password) =>
      _authRepository.login(email, password).then(
        (result) => result.fold(
          left,
          (user) {
            _userStore.setUser(user);
            return right(unit);
          },
        ),
      );
}
```

Rules:
- One use case per business operation
- `execute(...)` is the single public method
- Always inject repository interface type, not implementation
- Return `Either<SpecificFailure, SuccessType>` — never throw
- No UI logic, no navigation, no BuildContext

---

## Step 9 — Feature Files (5-file Pattern)

Each screen lives in `lib/features/<feature_name>/` with exactly 5 files.

### FILE 1: `<feature>_initial_params.dart`

Data only — no navigation callbacks.

```dart
// Data-only params
class ProductDetailInitialParams {
  final String productId;
  const ProductDetailInitialParams({required this.productId});
}

// Data callback allowed — returns a result, not a navigation action
class EditProductInitialParams {
  final String productId;
  final void Function(Product product) onProductUpdated;
  const EditProductInitialParams({
    required this.productId,
    required this.onProductUpdated,
  });
}
```

When different callers need different post-action behavior, use a **data enum** instead of a callback:

```dart
enum PostSignInIntent { viewHome, continueCheckout }

class SignInInitialParams {
  final PostSignInIntent intent;
  const SignInInitialParams({this.intent = PostSignInIntent.viewHome});
}
```

### FILE 2: `<feature>_state.dart`

Immutable snapshot of everything the UI needs.

```dart
import 'package:PACKAGE_NAME/core/domain/models/product.dart';
import '<feature>_initial_params.dart';

class ProductDetailState {
  final Product product;
  final bool isLoading;
  final bool isDeleting;

  bool get canEdit => product.id.isNotEmpty && !isLoading;

  const ProductDetailState({
    required this.product,
    required this.isLoading,
    required this.isDeleting,
  });

  factory ProductDetailState.initial({
    required ProductDetailInitialParams initialParams,
  }) => const ProductDetailState(
        product: Product.empty(),
        isLoading: false,
        isDeleting: false,
      );

  ProductDetailState copyWith({
    Product? product,
    bool? isLoading,
    bool? isDeleting,
  }) => ProductDetailState(
        product: product ?? this.product,
        isLoading: isLoading ?? this.isLoading,
        isDeleting: isDeleting ?? this.isDeleting,
      );
}
```

### FILE 3: `<feature>_cubit.dart`

Business logic. Calls **use cases only**. All navigation through `navigator`.

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:PACKAGE_NAME/core/domain/use_cases/get_product_use_case.dart';
import 'package:PACKAGE_NAME/core/domain/use_cases/delete_product_use_case.dart';
import '<feature>_initial_params.dart';
import '<feature>_navigator.dart';
import '<feature>_state.dart';

class ProductDetailCubit extends Cubit<ProductDetailState> {
  final ProductDetailInitialParams initialParams;
  final GetProductUseCase _getProductUseCase;
  final DeleteProductUseCase _deleteProductUseCase;
  final ProductDetailNavigator navigator;

  ProductDetailCubit(
    this.initialParams,
    this._getProductUseCase,
    this._deleteProductUseCase,
    this.navigator,
  ) : super(ProductDetailState.initial(initialParams: initialParams));

  void onInit() => _loadProduct();

  Future<void> _loadProduct() async {
    emit(state.copyWith(isLoading: true));
    final result = await _getProductUseCase.execute(initialParams.productId);
    result.fold(
      (failure) {
        emit(state.copyWith(isLoading: false));
        navigator.showError(failure.displayableFailure().message);
      },
      (product) => emit(state.copyWith(product: product, isLoading: false)),
    );
  }

  void onTapEdit() => navigator.openEditProduct(
        EditProductInitialParams(
          productId: state.product.id,
          onProductUpdated: (updated) => emit(state.copyWith(product: updated)),
        ),
      );

  Future<void> onTapDelete() async {
    emit(state.copyWith(isDeleting: true));
    final result = await _deleteProductUseCase.execute(state.product.id);
    result.fold(
      (failure) {
        emit(state.copyWith(isDeleting: false));
        navigator.showError(failure.displayableFailure().message);
      },
      (_) => navigator.close(),
    );
  }
}
```

### FILE 4: `<feature>_navigator.dart`

Two parts in one file: the **navigator class** (used by the cubit) and the **route mixin** (used by other navigators to open this screen).

```dart
import 'package:get_it/get_it.dart';
import 'package:PACKAGE_NAME/features/edit_product/edit_product_navigator.dart';
import 'package:PACKAGE_NAME/navigation/app_navigator.dart';
import 'package:PACKAGE_NAME/navigation/close_route.dart';
import 'package:PACKAGE_NAME/navigation/error_dialog_route.dart';
import '<feature>_initial_params.dart';
import '<feature>_page.dart';

// Part 1: Navigator class — mixes in routes for every screen THIS cubit can navigate to
class ProductDetailNavigator with CloseRoute, ErrorDialogRoute, EditProductRoute {
  ProductDetailNavigator(this.appNavigator);

  @override
  final AppNavigator appNavigator;
}

// Part 2: Route mixin — used by OTHER navigators that want to open THIS screen
mixin ProductDetailRoute {
  Future<void> openProductDetail(ProductDetailInitialParams initialParams) {
    return appNavigator.push(
      materialRoute(GetIt.instance<ProductDetailPage>(param1: initialParams)),
    );
  }

  AppNavigator get appNavigator;
}
```

Rules:
- Always include `CloseRoute` and `ErrorDialogRoute` in the navigator class
- Add one route mixin for every screen the cubit can navigate to
- Circular imports between navigator files are fine — they're mixin-only
- The route mixin uses `GetIt.instance<Page>(param1: params)` to instantiate the page

### FILE 5: `<feature>_page.dart`

UI widget. Receives cubit by constructor, uses `BlocBuilder` for all state reads.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '<feature>_cubit.dart';
import '<feature>_state.dart';

class ProductDetailPage extends StatefulWidget {
  final ProductDetailCubit cubit;

  const ProductDetailPage({Key? key, required this.cubit}) : super(key: key);

  @override
  State<ProductDetailPage> createState() => _ProductDetailPageState();
}

class _ProductDetailPageState extends State<ProductDetailPage> {
  ProductDetailCubit get cubit => widget.cubit;

  @override
  void initState() {
    super.initState();
    cubit.onInit();
  }

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<ProductDetailCubit, ProductDetailState>(
      bloc: cubit,
      builder: (context, state) {
        return Scaffold(
          appBar: AppBar(
            title: Text(state.product.title),
            actions: [
              if (state.canEdit)
                IconButton(
                  onPressed: cubit.onTapEdit,
                  icon: const Icon(Icons.edit),
                ),
            ],
          ),
          body: state.isLoading
              ? const Center(child: CircularProgressIndicator())
              : _buildContent(state),
        );
      },
    );
  }

  Widget _buildContent(ProductDetailState state) {
    return Column(
      children: [
        Text(state.product.title),
        ElevatedButton(
          onPressed: state.isDeleting ? null : cubit.onTapDelete,
          child: state.isDeleting
              ? const CircularProgressIndicator()
              : const Text('Delete'),
        ),
      ],
    );
  }
}
```

Page rules (non-negotiable):
- All `onPressed`/`onTap`/`onChanged` point directly to a cubit method — never inline logic
- Read all display values from `state` inside `BlocBuilder`
- No `setState` calls anywhere
- No `context.read`, `context.watch`, `Provider.of`, or direct store/repo access
- Navigation is triggered by the cubit via `navigator` — never call `Navigator.push` from the page

---

## Step 10 — Shared Navigation Infrastructure

Create these files if they don't exist.

### `lib/navigation/app_navigator.dart`

```dart
import 'package:flutter/material.dart';

class AppNavigator {
  static final navigatorKey = GlobalKey<NavigatorState>();

  Future<R?> push<R>(Route<R> route, {bool useRoot = false}) async =>
      _navigator(useRoot: useRoot).push(route);

  Future<R?> pushReplacement<R>(Route<R> route, {bool useRoot = false, R? result}) async =>
      _navigator(useRoot: useRoot).pushReplacement(route, result: result);

  void close() => closeWithResult(null);
  void closeWithResult<T>(T result) => _navigator().pop(result);
  void popUntilRoot() => _navigator().popUntil((route) => route.isFirst);
}

Route<T> materialRoute<T>(Widget page, {bool fullScreenDialog = false}) =>
    MaterialPageRoute(builder: (context) => page, fullscreenDialog: fullScreenDialog);

Route<T> fadeInRoute<T>(Widget page, {int? durationMillis}) => PageRouteBuilder<T>(
      transitionDuration: Duration(milliseconds: durationMillis ?? 250),
      pageBuilder: (context, animation, _) =>
          FadeTransition(opacity: animation, child: page),
    );

Route<T> slideBottomRoute<T>(Widget page, {int? durationMillis, bool fullScreenDialog = false}) =>
    PageRouteBuilder<T>(
      transitionDuration: Duration(milliseconds: durationMillis ?? 250),
      fullscreenDialog: fullScreenDialog,
      pageBuilder: (context, animation, _) => SlideTransition(
        position: Tween<Offset>(begin: const Offset(0, 1), end: Offset.zero)
            .animate(CurvedAnimation(parent: animation, curve: Curves.easeOutQuint)),
        child: page,
      ),
    );

Route<T> bottomSheetRoute<T>(Widget page) => ModalBottomSheetRoute(
      builder: (context) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom),
        child: page,
      ),
      isScrollControlled: true,
      useSafeArea: true,
      shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.vertical(top: Radius.circular(10))),
    );

NavigatorState _navigator({bool useRoot = false}) => AppNavigator.navigatorKey.currentState!;
```

### `lib/navigation/close_route.dart`

```dart
import 'package:PACKAGE_NAME/navigation/app_navigator.dart';

mixin CloseRoute {
  void close() => appNavigator.close();
  AppNavigator get appNavigator;
}
```

### `lib/navigation/error_dialog_route.dart`

```dart
import 'package:flutter/material.dart';
import 'package:PACKAGE_NAME/navigation/app_navigator.dart';

mixin ErrorDialogRoute {
  void showError(String message) {
    final context = AppNavigator.navigatorKey.currentContext!;
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Error'),
        content: Text(message),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: const Text('OK'),
          ),
        ],
      ),
    );
  }

  AppNavigator get appNavigator;
}
```

---

## Step 11 — Dependency Injection

Register every layer in the correct order: infrastructure → stores → repositories → use cases → features.

```dart
import 'package:get_it/get_it.dart';
final getIt = GetIt.instance;

Future<void> init() async {
  // --- Infrastructure ---
  getIt.registerLazySingleton(() => AppNavigator());

  // --- Global Stores ---
  getIt.registerLazySingleton(() => UserStore());

  // --- Repositories (interface → implementation) ---
  getIt.registerLazySingleton<AuthRepository>(() => FirebaseAuthRepository(getIt()));
  getIt.registerLazySingleton<ProductRepository>(() => FirebaseProductRepository());

  // --- Use Cases ---
  getIt.registerSingleton(LoginUseCase(getIt(), getIt()));
  getIt.registerSingleton(GetProductUseCase(getIt()));
  getIt.registerSingleton(DeleteProductUseCase(getIt()));

  // --- Feature: product_detail ---
  getIt.registerFactory(() => ProductDetailNavigator(getIt()));
  getIt.registerFactoryParam<ProductDetailCubit, ProductDetailInitialParams, void>(
    (params, _) => ProductDetailCubit(params, getIt(), getIt(), getIt()),
  );
  getIt.registerFactoryParam<ProductDetailPage, ProductDetailInitialParams, void>(
    (params, _) => ProductDetailPage(cubit: getIt(param1: params)),
  );

  // ... repeat for every feature
}
```

Registration rules:

| Type | Method | Reason |
|------|--------|--------|
| `AppNavigator` | `registerLazySingleton` | One instance, created on first use |
| Stores | `registerLazySingleton` | App-wide state, single instance |
| Repository interfaces | `registerLazySingleton` | Stateless data access, one instance |
| Use cases | `registerSingleton` | Stateless, created eagerly at startup |
| Feature Navigators | `registerFactory` | New instance per screen |
| Feature Cubits | `registerFactoryParam` | New instance per screen, needs `InitialParams` |
| Feature Pages | `registerFactoryParam` | New instance per screen, needs `InitialParams` |

Wire the navigator key into `MaterialApp`:

```dart
MaterialApp(
  navigatorKey: AppNavigator.navigatorKey,
  home: getIt<HomePage>(param1: const HomeInitialParams()),
)
```

---

## Step 12 — Remove Navigation Callback Anti-Patterns

For every feature, audit `InitialParams` for navigation callbacks and eliminate them.

### 12A — Identify what needs to change

For each feature:
1. Read `<feature>_initial_params.dart` — list every callback that is a navigation action.
2. Read `<feature>_cubit.dart` — identify every `initialParams.onXxx?.call()` navigation call.
3. Read `<feature>_navigator.dart` — note which route mixins it already has.
4. Search the whole codebase for every place `<Feature>InitialParams(...)` is constructed.

Record: which callbacks to remove, which route mixins to add, which use cases the cubit needs to load data before navigating.

### 12B — Remove navigation callbacks from InitialParams

```dart
// BEFORE
class RecordingInitialParams {
  final String userId;
  final void Function(Entry) onRecordingComplete; // data callback — KEEP
  final void Function()? onViewEntries;           // navigation callback — REMOVE
  final void Function()? onSignIn;                // navigation callback — REMOVE
}

// AFTER
class RecordingInitialParams {
  final String userId;
  final void Function(Entry) onRecordingComplete; // data callback — KEEP
}
```

If callers need different post-action behavior, replace callbacks with a data enum:

```dart
enum PostSignInIntent { viewHome, continueRecording }

class SignInInitialParams {
  final PostSignInIntent intent;
  const SignInInitialParams({this.intent = PostSignInIntent.viewHome});
}
```

### 12C — Add route mixins to the navigator

```dart
// BEFORE
class RecordingNavigator with CloseRoute, ErrorDialogRoute {
  RecordingNavigator(this.appNavigator);
  @override final AppNavigator appNavigator;
}

// AFTER — added EntriesListRoute because cubit now navigates there
class RecordingNavigator with CloseRoute, ErrorDialogRoute, EntriesListRoute {
  RecordingNavigator(this.appNavigator);
  @override final AppNavigator appNavigator;
}
```

Import the target's navigator file to bring in its mixin:
```dart
import 'package:PACKAGE_NAME/features/entries_list/entries_list_navigator.dart';
```

### 12D — Rewrite cubit methods

Replace every `initialParams.onXxx?.call()` navigation call with `navigator.openXxx(...)`.

```dart
// BEFORE
void onTapViewEntries() => initialParams.onViewEntries?.call();

// AFTER — cubit loads data itself, then navigates
Future<void> onTapViewEntries() async {
  final result = await _getEntriesUseCase.execute(initialParams.userId);
  final entries = result.fold((_) => <Entry>[], (e) => e);
  navigator.openEntriesList(EntriesListInitialParams(entries: entries));
}
```

For closing the current screen:
```dart
void onTapBack() => navigator.close(); // from CloseRoute
```

For popping all the way to root:
```dart
void onTapDone() => navigator.appNavigator.popUntilRoot();
```

### 12E — Update DI if the cubit gained new parameters

The `getIt()` call count must match the cubit constructor parameter count exactly.

```dart
// BEFORE — cubit had 4 deps
getIt.registerFactoryParam<RecordingCubit, RecordingInitialParams, void>(
  (params, _) => RecordingCubit(params, getIt(), getIt(), getIt(), getIt()),
);

// AFTER — added GetEntriesUseCase
getIt.registerFactoryParam<RecordingCubit, RecordingInitialParams, void>(
  (params, _) => RecordingCubit(params, getIt(), getIt(), getIt(), getIt(), getIt()),
);
```

### 12F — Fix every call site of the changed InitialParams

Search the codebase for every constructor call of the changed `InitialParams`. Remove the deleted callback arguments. Delete any dead helper methods that only existed to wire those callbacks.

```dart
// BEFORE — bootstrap wired navigation callbacks
navigator.openRecording(RecordingInitialParams(
  userId: userStore.userId,
  onRecordingComplete: _handleComplete,
  onViewEntries: _openEntriesList,  // ← delete
));

// AFTER — only data
navigator.openRecording(RecordingInitialParams(
  userId: userStore.userId,
  onRecordingComplete: _handleComplete,
));
```

Also update any Page that referenced a removed param field — visibility decisions must be based on data state, not callback presence:

```dart
// BEFORE
if (cubit.initialParams.isSignedIn && cubit.initialParams.onViewEntries != null)

// AFTER — button visibility based on data only
if (cubit.initialParams.isSignedIn)
```

---

## Step 13 — Migrate Existing Logic

For each existing feature or screen:

1. **Identify the data** it fetches/writes → create domain model, JSON model, repository interface + implementation, failure type, and use case.
2. **Identify the state** it manages → move all local widget state, `setState` calls, and Provider/Riverpod state into the new `FeatureState`.
3. **Identify the actions** → move all methods to the cubit, wiring them to use cases.
4. **Identify the navigation** → move all `Navigator.push`, `context.go`, etc. into the feature navigator. Remove all navigation callbacks from `InitialParams`.
5. **Rewrite the UI** using `BlocBuilder`.
6. **Delete old files** completely.

Do not leave old and new code coexisting. Fully migrate each feature before moving to the next.

---

## Step 14 — Validate

```bash
flutter analyze lib/
```

Acceptable: `info` items (`use_super_parameters`, `deprecated_member_use`). Not acceptable: any `error` or `warning`.

Common errors to watch for:
- `undefined_getter` — a Page still references a removed `InitialParams` field
- `unused_element` — a dead method in a bootstrap or parent coordinator
- `Too many positional arguments` — DI `getIt()` count doesn't match the cubit constructor
- `The getter 'xxx' isn't defined` — forgot to add a route mixin to the navigator

Fix all errors. Re-run `flutter analyze` until clean.

---

## Final Checklist

**Domain layer**
- [ ] Every entity extends `Equatable`, has `empty()`, `copyWith`, `final` fields
- [ ] Every repository is an `abstract class` in `domain/repositories/`
- [ ] Every use case only imports domain types (no Firebase, no `_json.dart`)
- [ ] Every failure implements `HasDisplayableFailure` with a typed enum

**Data layer**
- [ ] Every JSON model has `fromJson`, `toJson`, `toDomain()`
- [ ] Every repository implementation `implements` the domain interface
- [ ] All external API calls inside `try/catch` returning `left(Failure)` on error
- [ ] No domain models contain `fromJson`/`toJson`

**Features layer**
- [ ] Every feature has exactly 5 files
- [ ] Every `InitialParams` has zero navigation callbacks — data and data-returning callbacks only
- [ ] Context-dependent post-action behavior uses an intent enum, not callbacks
- [ ] Every State has `factory State.initial(...)` and `copyWith`
- [ ] Every Page calls `cubit.onInit()` in `initState`
- [ ] Every Page uses only `BlocBuilder` — no `setState`, no `context.read`, no direct store/repo access
- [ ] Every Cubit injects use cases only (not repositories or external APIs)
- [ ] Every Cubit calls `navigator.openXxx(...)` for navigation — no `initialParams.onXxx?.call()`
- [ ] Every Navigator class `with`-mixes a route mixin for every screen the cubit navigates to
- [ ] Every Navigator class has its own route mixin defined in the same file

**DI**
- [ ] Repositories: `registerLazySingleton<Interface>(() => Implementation())`
- [ ] Use cases: `registerSingleton`
- [ ] Navigators: `registerFactory`
- [ ] Cubits and Pages: `registerFactoryParam`
- [ ] `getIt()` call count in each `registerFactoryParam` matches the cubit constructor parameter count
- [ ] `MaterialApp` has `navigatorKey: AppNavigator.navigatorKey`

**Dependencies**
- [ ] `pubspec.yaml` includes: `flutter_bloc`, `get_it`, `equatable`, `dartz`
- [ ] All old state management removed (`provider`, `riverpod`, raw `setState` for business logic, etc.)
