# Riverpod Code Snippets

Use these snippets as templates for common Riverpod implementations.

## 1. Async Notifier (Standard Data Fetching)

For fetching data from an API and managing its state.

```dart
@riverpod
class UserList extends _$UserList {
  @override
  FutureOr<List<User>> build() async {
    // 1. Fetch data
    final repository = ref.watch(userRepositoryProvider);
    return repository.getUsers();
  }

  // 2. Methods for mutation (side effects)
  Future<void> addUser(User user) async {
    final link = ref.keepAlive();
    
    // Optional: Show loading indicator while keeping previous data
    state = const AsyncLoading().copyWithPrevious(state);
    
    try {
      final repository = ref.read(userRepositoryProvider);
      await repository.addUser(user);
      
      // Success: Refresh data
      ref.invalidateSelf();
      await future; // Wait for the refresh to complete
    } catch (e) {
      // Failure: Do NOT set state = AsyncError, as that wipes the list.
      // Instead, rethrow so the UI (button) can catch it and show a Snackbar.
      // Or handle it specifically.
      rethrow;
    } finally {
      link.close();
    }
  }
}
```

## 2. Simple Provider (Read-Only)

For simple values or dependencies.

```dart
@riverpod
String appTitle(Ref ref) {
  return 'My App';
}
```

## 3. Family Provider (With Arguments)

For providers that need external parameters.

```dart
@riverpod
Future<User> user(Ref ref, {required int id}) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.getUser(id);
}
```

## 4. Widget Consumer (Watching State)

How to consume providers in the UI.

```dart
class UserListScreen extends ConsumerWidget {
  const UserListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 1. Watch the provider
    final asyncUsers = ref.watch(userListProvider);

    // 2. Handle states with switch expression
    return Scaffold(
      body: switch (asyncUsers) {
        AsyncData(:final value) => ListView.builder(
            itemCount: value.length,
            itemBuilder: (context, index) => UserTile(value[index]),
          ),
        AsyncError(:final error) => Center(child: Text('Error: $error')),
        _ => const Center(child: CircularProgressIndicator()),
      },
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // 3. Trigger mutation
          ref.read(userListProvider.notifier).addUser(newUser);
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

## 5. Side Effects (Action Handling)

Handling one-off actions like navigation or showing snackbars based on state changes.

```dart
ref.listen(userListProvider, (previous, next) {
  if (next is AsyncError) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(next.error.toString())),
    );
  } else if (next is AsyncData && previous is! AsyncData) {
     // Navigate on success
     context.go('/home');
  }
});
```

## 6. Ref.mounted Check

Preventing state updates on disposed providers.

```dart
Future<void> longRunningTask() async {
  state = const AsyncLoading();
  try {
    final result = await _api.fetch();
    // Check if the provider is still mounted before updating state
    if (ref.mounted) {
      state = AsyncData(result);
    }
  } catch (e, st) {
    if (ref.mounted) {
      state = AsyncError(e, st);
    }
  }
}
```
