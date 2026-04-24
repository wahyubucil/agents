# Testing Riverpod

Best practices for testing Riverpod providers and widgets.

## 1. Unit Testing Providers

Use `ProviderContainer` to test providers in isolation.

```dart
void main() {
  test('UserList provider returns initial data', () async {
    // 1. Create a container
    final container = ProviderContainer(
      overrides: [
        // Mock dependencies
        userRepositoryProvider.overrideWithValue(MockUserRepository()),
      ],
    );
    addTearDown(container.dispose);

    // 2. Read the provider
    final userList = container.read(userListProvider);

    // 3. Assert initial state (e.g. loading or data)
    expect(userList, isA<AsyncLoading>());
    
    // 4. Wait for async value
    await expectLater(
      container.read(userListProvider.future),
      completion(isEmpty),
    );
  });
}
```

## 2. Widget Testing

Use `ProviderScope` to wrap widgets in tests.

```dart
void main() {
  testWidgets('UserListScreen displays loading initially', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          userListProvider.overrideWith((ref) => AsyncValue.data([])), // Mock data
        ],
        child: const MaterialApp(home: UserListScreen()),
      ),
    );

    expect(find.byType(CircularProgressIndicator), findsNothing); // Because we mocked data
    expect(find.byType(ListView), findsOneWidget);
  });
}
```

## 3. Mocks with Mocktail

Standard mocking pattern.

```dart
import 'package:mocktail/mocktail.dart';

class MockUserRepository extends Mock implements UserRepository {}

final mockRepo = MockUserRepository();
when(() => mockRepo.getUsers()).thenAnswer((_) async => [User(id: 1, name: 'Test')]);
```
