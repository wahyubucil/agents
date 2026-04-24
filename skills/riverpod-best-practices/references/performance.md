# Performance Optimization

Techniques to minimize rebuilds and optimize Riverpod performance.

## 1. Select Specific Fields (`ref.select`)

Avoid watching large objects if you only need one field.

```dart
// ❌ BAD: Rebuilds on ANY product change (price, name, description...)
final product = ref.watch(productProvider);
return Text('\$${product.price}');

// ✅ GOOD: Only rebuilds when 'price' changes
final price = ref.watch(productProvider.select((p) => p.price));
return Text('\$$price');
```

## 2. Filter List Updates

Rebuild only when the *count* changes, not when items are modified.

```dart
// Rebuilds only if the length of the list changes
final count = ref.watch(todoListProvider.select((todos) => todos.length));
```

## 3. Avoid Watching in Loops

Don't `watch` providers inside a loop or `ListView.builder` directly if it depends on the index in a way that causes all items to rebuild.

**Better Pattern:** Pass the ID to a child widget, and have the child widget `watch` the specific item provider.

```dart
// Parent
ListView.builder(
  itemCount: ids.length,
  itemBuilder: (context, index) => TodoItem(id: ids[index]),
);

// Child (Optimized)
class TodoItem extends ConsumerWidget {
  final String id;
  ...
  build(context, ref) {
    // Only this widget rebuilds if this specific Todo changes
    final todo = ref.watch(todoProvider(id)); 
    return Text(todo.title);
  }
}
```

## 4. `ref.read` vs `ref.watch`

*   **`ref.watch`**: Use in `build()` to listen for changes.
*   **`ref.read`**: Use in **events** (onPressed, callbacks) to read one-time values.
    *   **WARNING:** NEVER use `ref.read` inside `build()`. It won't update the UI when data changes.

## 5. Caching and `keepAlive`

Prevent expensive re-fetching by keeping providers alive.

```dart
// Cache forever (e.g. AppConfig)
@Riverpod(keepAlive: true)
Future<Config> config(Ref ref) => ...

// Cache for 5 minutes (Auto-dispose with timer)
@riverpod
Future<Data> cachedData(Ref ref) async {
  final link = ref.keepAlive();
  // Close link after 5 mins, allowing disposal if no listeners
  final timer = Timer(const Duration(minutes: 5), () => link.close());
  ref.onDispose(() => timer.cancel());
  
  return fetchData();
}
```
