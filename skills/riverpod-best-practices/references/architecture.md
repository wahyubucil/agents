# Repository & Architecture Patterns

Structure your Riverpod app using a 3-layer modular architecture.

## Architecture Layers

### 1. Data Layer (Repositories)
Handles data fetching (API, DB). independent of Riverpod state.
*   **Location:** `lib/api/repositories/` or `lib/modules/[feature]/repositories/`
*   **Role:** Fetch data, map to entities.

```dart
@riverpod
TodoRepository todoRepository(Ref ref) {
  return TodoRepository(dio: ref.watch(dioProvider));
}

class TodoRepository {
  final Dio dio;
  TodoRepository({required this.dio});

  Future<List<Todo>> fetchTodos() async { ... }
}
```

### 2. Domain/Application Layer (Notifiers)
Manages state and business logic.
*   **Location:** `lib/modules/[feature]/providers/`
*   **Role:** Call repositories, manage loading/error states, expose data to UI.

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    return ref.watch(todoRepositoryProvider).fetchTodos();
  }
  
  // Business logic / Mutations
  Future<void> add(String title) async { ... }
}
```

### 3. Presentation Layer (Screens/Widgets)
Displays data.
*   **Location:** `lib/modules/[feature]/screens/`
*   **Role:** Watch providers, display UI, trigger events.

```dart
class TodoScreen extends ConsumerWidget {
  build(context, ref) {
    final todos = ref.watch(todoListProvider);
    return todos.when( ... );
  }
}
```

## Common Anti-Patterns

### ❌ Multiple Sources of Truth
Don't mix local state (`setState`) with Provider state for the same data.

```dart
// BAD
class MyWidget extends StatefulWidget {
  List<Todo> todos = []; // Local copy??
  
  build() {
    final providerTodos = ref.watch(todosProvider); 
    // Now you have two lists. Which one is real?
  }
}
```

### ❌ Not Invalidating Dependencies (Safe Pattern)
When a user logs out, ensure you invalidate or dispose user-specific data. **Note:** Always check `ref.mounted` if this follows an async call.

```dart
Future<void> logout(Ref ref) async {
  // 1. Keep alive if this logic is critical and shouldn't be interrupted
  final link = ref.keepAlive(); 
  
  try {
    final authRepo = ref.read(authRepoProvider);
    await authRepo.logout();
    
    // 2. Safe to use ref because we kept it alive
    // Force refresh of user-dependent providers
    ref.invalidate(userProfileProvider); 
    ref.invalidate(ordersProvider);
  } finally {
    link.close();
  }
}
```

### ❌ Bad "Cleanup"
If you use a Stream or Controller in a provider, always dispose it.

```dart
@riverpod
Stream<int> myStream(Ref ref) {
  final controller = StreamController<int>();
  
  // ✅ Clean up when provider is destroyed
  ref.onDispose(() => controller.close());
  
  return controller.stream;
}
```
