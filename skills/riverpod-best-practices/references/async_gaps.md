# Handling Async Gaps and Provider Lifecycle in Riverpod v3

Riverpod v3 enforces stricter lifecycle management than v2. This guide explains how to avoid common pitfalls like `UnmountedRefException` and unexpected provider disposal.

## The "Unmounted Ref" Problem

In Riverpod v3, `autoDispose` providers are disposed **immediately** when they have no listeners.
*   **Crucial:** `ref.read()` does **NOT** count as a listener.
*   If you `await ref.read(provider.future)`, the provider starts, yields the future, and if nothing else is watching it, **Riverpod disposes of it immediately**, even if the future hasn't completed yet.

### Common Symptoms
*   `UnmountedRefException` when using `ref` after an `await`.
*   Providers disposing in the middle of a function execution.
*   "Ref is disposed before Future-provider completes" errors.

## Solutions

### 1. The `ref.keepAlive()` Pattern (Recommended)

If a provider's operation is critical and must complete once started (e.g., network requests, file writes), explicitly keep it alive during the operation.

```dart
@riverpod
Future<String> submitData(Ref ref) async {
  // 1. Prevent disposal even if listeners disappear
  final link = ref.keepAlive();

  try {
    // 2. Perform async work
    await Future.delayed(const Duration(seconds: 2));
    
    // It is now safe to use 'ref' here because we forced it to stay alive
    ref.invalidate(otherProvider);
    
    return "Success";
  } finally {
    // 3. IMPORTANT: Release the link so the provider can be disposed later
    link.close();
  }
}
```

### 2. The `ref.mounted` Check

If you are just reading data and don't strictly need to keep the provider alive, you **must** check if the ref is still valid after any `await`.

```dart
Future<void> onButtonPress(WidgetRef ref) async {
  // 1. Trigger async action
  final result = await ref.read(myProvider.future);
  
  // 2. Async gap happens here... provider might be disposed!

  // 3. CHECK if the ref is still valid before using it
  if (!ref.context.mounted) return; // For WidgetRef
  // OR
  // if (!ref.mounted) return; // For Provider Ref

  // 4. Safe to use
  ref.read(otherProvider.notifier).update(result);
}
```

### 3. Prefer `watch`/`listen`

The "Riverpod Way" is to use `ref.watch` (in build) or `ref.listen` (in body/build), which creates a subscription that guarantees the provider stays alive.

```dart
// ✅ CORRECT: ref.watch keeps 'myFutureProvider' alive
final asyncValue = ref.watch(myFutureProvider);
```

---

## Deep Dive: Analysis of Github Issues

Understanding the "Why" behind these changes (based on Issues #4096, #3869, #4325).

### 1. Issue #4096: "Can not use ref in a notifier after async gap"
*   **The Problem:** Using `ref.read` inside a Notifier's method after an `await`. By the time the `await` finishes, the provider might be disposed.
*   **Key Insight:** This is **expected behavior** in v3. If a provider is no longer being listened to, it is disposed to save resources. Modifying a dead object is a bug.
*   **Takeaway:** Check `ref.mounted` or use `keepAlive`.

### 2. Issue #4325: "ref is disposed before Future-provider completes"
*   **The Problem:** Running `await ref.read(provider.future)`. The provider starts, hits an `await` inside itself, and immediately disposes because `read` is not a listener.
*   **Key Insight:** `ref.read` is for obtaining a value, not for keeping a provider alive.
*   **Maintainer's Stance:** If you need the provider to finish a task, you must explicitly keep it alive using the pattern in Solution #1.

### 3. Issue #3869: Linting against async gaps
*   **Context:** The community is moving towards strict linting rules (similar to `use_build_context_synchronously`) to prevent using `Ref` across async gaps without checks.
