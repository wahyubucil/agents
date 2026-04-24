# Provider Selection Guide

Choosing the right provider type for your use case.

## Quick Decision Tree

### 1. Immutable / Computed Values → `Provider`
Use for values that don't change (like config) or values computed synchronously from other providers.

```dart
@riverpod
String apiKey(Ref ref) => 'YOUR_API_KEY';

@riverpod
double totalPrice(Ref ref) {
  final cart = ref.watch(cartProvider);
  return cart.items.fold(0, (sum, item) => sum + item.price);
}
```

### 2. Simple Synchronous State → `Notifier`
Use for state that changes synchronously (counters, ui toggles, form state) and doesn't need `Future`/`Stream`.

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
}
```

### 3. Async Data (Fetching) → `FutureProvider` (Functional)
Use for simple data fetching where you don't need methods to mutate the state (read-only from UI perspective).

```dart
@riverpod
Future<List<User>> users(Ref ref) {
  return ref.watch(userRepositoryProvider).fetchUsers();
}
```

### 4. Async Data with Mutations → `AsyncNotifier` (Class-based)
**PREFERRED for most features.** Use when you need to fetch data AND perform actions (add, update, delete) on that data.

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    return ref.watch(todoRepositoryProvider).fetchTodos();
  }

  Future<void> addTodo(String title) async {
    final link = ref.keepAlive();
    state = const AsyncLoading().copyWithPrevious(state);
    
    try {
      await ref.read(todoRepositoryProvider).createTodo(title);
      // Success: Refresh data
      ref.invalidateSelf();
      await future; 
    } catch (e) {
      // Failure: Rethrow or handle
      rethrow;
    } finally {
      link.close();
    }
  }
}
```

### 5. Real-time Streams → `StreamProvider`
Use for data that comes from a stream (Firebase, WebSockets).

```dart
@riverpod
Stream<User?> authState(Ref ref) {
  return FirebaseAuth.instance.authStateChanges();
}
```

## Summary Table

| Use Case | Provider Type | Annotation |
| :--- | :--- | :--- |
| Read-only value | `Provider` | `@riverpod` (func) |
| Sync State (Counter) | `Notifier` | `@riverpod` (class) |
| Simple Fetch | `FutureProvider` | `@riverpod` (func) |
| **Fetch + Mutate** | **`AsyncNotifier`** | **`@riverpod` (class)** |
| Streams | `StreamProvider` | `@riverpod` (func) |
