# Standard Operating Procedure (SOP): Dio Package Usage in Flutter

**Document Version:** 1.0
**Applies to:** Flutter projects using `dio` for HTTP networking
**Owner:** Mobile/App Engineering Team

---

## 1. Purpose

This SOP defines the standard approach for integrating, configuring, and using the `dio` package in Flutter applications to ensure consistency, maintainability, security, and testability across the codebase.

## 2. Scope

Applies to all Flutter projects (mobile, web, desktop) that communicate with REST/GraphQL APIs via HTTP. Covers setup, configuration, interceptors, error handling, file upload/download, testing, and security.

---

## 3. Prerequisites

- Flutter SDK installed and project initialized.
- Basic understanding of Dart async/await and Futures.
- API base URL(s) and authentication scheme documented (API key, Bearer token, OAuth, etc.).

---

## 4. Installation

### 4.1 Add Dependency

```bash
flutter pub add dio
```

This adds the latest stable version to `pubspec.yaml`:

```yaml
dependencies:
  dio: ^5.7.0
```

### 4.2 Optional Companion Packages

| Package | Purpose |
|---|---|
| `pretty_dio_logger` | Readable request/response logging in debug mode |
| `dio_smart_retry` | Automatic retry with backoff |
| `dio_cache_interceptor` | HTTP caching |
| `connectivity_plus` | Network connectivity checks before requests |
| `flutter_secure_storage` | Secure token storage for auth headers |

Install as needed:

```bash
flutter pub add pretty_dio_logger dio_smart_retry
```

---

## 5. Project Structure (Recommended)

```
lib/
├── core/
│   └── network/
│       ├── dio_client.dart          # Singleton Dio instance + config
│       ├── interceptors/
│       │   ├── auth_interceptor.dart
│       │   ├── logging_interceptor.dart
│       │   └── error_interceptor.dart
│       ├── api_exception.dart       # Custom exception model
│       └── network_result.dart      # Result/Either wrapper (optional)
├── data/
│   └── datasources/
│       └── remote/
│           └── user_remote_datasource.dart
```

Keep all Dio configuration centralized in one place (`dio_client.dart`). Do **not** instantiate `Dio()` ad hoc across the app.

---

## 6. Step-by-Step Setup

### 6.1 Create a Centralized Dio Client

```dart
// lib/core/network/dio_client.dart
import 'package:dio/dio.dart';

class DioClient {
  DioClient._internal();
  static final DioClient _instance = DioClient._internal();
  factory DioClient() => _instance;

  late final Dio dio = _createDio();

  Dio _createDio() {
    final dio = Dio(
      BaseOptions(
        baseUrl: 'https://api.example.com/v1',
        connectTimeout: const Duration(seconds: 15),
        receiveTimeout: const Duration(seconds: 15),
        sendTimeout: const Duration(seconds: 15),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        // Avoid throwing on non-2xx by default if you want custom handling
        // validateStatus: (status) => status != null && status < 500,
      ),
    );

    dio.interceptors.addAll([
      AuthInterceptor(),
      ErrorInterceptor(),
      if (kDebugMode) LoggingInterceptor(),
    ]);

    return dio;
  }
}
```

**Rules:**
- Use a **singleton** pattern (or dependency injection via `get_it` / `riverpod` / `provider`) — never create multiple ad hoc `Dio()` instances per screen.
- Set explicit timeouts. Never leave them at default/infinite.
- Set `baseUrl` centrally; never hardcode full URLs in feature code.

### 6.2 Register with Dependency Injection (Recommended)

```dart
// Using get_it
final sl = GetIt.instance;

void setupLocator() {
  sl.registerLazySingleton<Dio>(() => DioClient().dio);
}
```

This makes Dio mockable for unit tests.

---

## 7. Interceptors

Interceptors are the backbone of consistent Dio usage. Standardize on three categories:

### 7.1 Auth Interceptor

```dart
class AuthInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await SecureStorage.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Attempt token refresh, then retry original request
      final refreshed = await AuthRepository.refreshToken();
      if (refreshed) {
        final opts = err.requestOptions;
        final newToken = await SecureStorage.getAccessToken();
        opts.headers['Authorization'] = 'Bearer $newToken';
        final response = await Dio().fetch(opts);
        return handler.resolve(response);
      }
    }
    handler.next(err);
  }
}
```

### 7.2 Logging Interceptor (Debug Only)

```dart
LogInterceptor(
  requestBody: true,
  responseBody: true,
  logPrint: (obj) => debugPrint(obj.toString()),
)
```

**Rule:** Never enable verbose body logging in release builds — risk of leaking PII/tokens in logs.

### 7.3 Error Normalization Interceptor

```dart
class ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final apiException = ApiException.fromDioError(err);
    handler.next(err.copyWith(error: apiException));
  }
}
```

**Interceptor ordering rule:** Add interceptors in the order they should execute for requests (auth → error → logging is a common order, since logging should see the final headers).

---

## 8. Error Handling

### 8.1 Define a Standard Exception Model

```dart
class ApiException implements Exception {
  final String message;
  final int? statusCode;
  final ApiErrorType type;

  ApiException({required this.message, this.statusCode, required this.type});

  factory ApiException.fromDioError(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return ApiException(message: 'Connection timed out', type: ApiErrorType.timeout);
      case DioExceptionType.badResponse:
        return ApiException(
          message: _extractServerMessage(error.response),
          statusCode: error.response?.statusCode,
          type: ApiErrorType.badResponse,
        );
      case DioExceptionType.connectionError:
        return ApiException(message: 'No internet connection', type: ApiErrorType.noInternet);
      case DioExceptionType.cancel:
        return ApiException(message: 'Request cancelled', type: ApiErrorType.cancelled);
      default:
        return ApiException(message: 'Unexpected error occurred', type: ApiErrorType.unknown);
    }
  }

  static String _extractServerMessage(Response? response) {
    try {
      return response?.data['message'] ?? 'Server error';
    } catch (_) {
      return 'Server error';
    }
  }
}

enum ApiErrorType { timeout, badResponse, noInternet, cancelled, unknown }
```

### 8.2 Rules

- **Always** wrap Dio calls in try/catch at the data-source layer — never let raw `DioException` propagate to the UI layer.
- Map every error to a typed, user-friendly message before it reaches the presentation layer.
- Distinguish network errors (no internet) from server errors (4xx/5xx) from client bugs (parsing failures).
- Never swallow exceptions silently (empty catch blocks are prohibited).

---

## 9. Making Requests (Standard Patterns)

### 9.1 GET

```dart
Future<List<User>> getUsers({int page = 1}) async {
  try {
    final response = await dio.get('/users', queryParameters: {'page': page});
    return (response.data['data'] as List)
        .map((json) => User.fromJson(json))
        .toList();
  } on DioException catch (e) {
    throw ApiException.fromDioError(e);
  }
}
```

### 9.2 POST

```dart
Future<User> createUser(UserDto dto) async {
  try {
    final response = await dio.post('/users', data: dto.toJson());
    return User.fromJson(response.data);
  } on DioException catch (e) {
    throw ApiException.fromDioError(e);
  }
}
```

### 9.3 File Upload (Multipart)

```dart
Future<void> uploadAvatar(String filePath) async {
  final formData = FormData.fromMap({
    'avatar': await MultipartFile.fromFile(filePath, filename: 'avatar.jpg'),
  });
  await dio.post('/users/avatar', data: formData);
}
```

### 9.4 File Download with Progress

```dart
Future<void> downloadFile(String url, String savePath, {
  void Function(int, int)? onProgress,
}) async {
  await dio.download(url, savePath, onReceiveProgress: onProgress);
}
```

### 9.5 Request Cancellation

```dart
final cancelToken = CancelToken();

dio.get('/search', queryParameters: {'q': query}, cancelToken: cancelToken);

// On dispose or new search:
cancelToken.cancel('Superseded by new request');
```

**Rule:** Always attach a `CancelToken` to requests tied to a widget lifecycle (e.g., search-as-you-type, screen navigation) and cancel it in `dispose()`.

---

## 10. Response Parsing & Models

- Use `json_serializable` / `freezed` for model classes — never manually parse nested JSON inline in the repository layer.
- Isolate parsing logic in `fromJson`/`toJson` factory constructors.
- For large payloads, consider parsing in a background isolate via `compute()` to avoid UI jank.

---

## 11. Environment Configuration

Maintain separate `baseUrl` and timeout configs per environment (dev/staging/prod) using `--dart-define` or a config file, never hardcoded conditionals scattered through the code:

```dart
const String baseUrl = String.fromEnvironment(
  'BASE_URL',
  defaultValue: 'https://api-dev.example.com/v1',
);
```

---

## 12. Security Guidelines

1. **Never** log full request/response bodies containing tokens, passwords, or PII in production.
2. Store tokens using `flutter_secure_storage`, not `SharedPreferences`.
3. Enforce HTTPS only; reject plain HTTP base URLs outside local dev.
4. For certificate pinning on sensitive apps (banking, health), configure `HttpClientAdapter` with pinned certificates.
5. Sanitize and validate all data before sending; do not trust client-side data blindly even for internal APIs.

---

## 13. Testing

### 13.1 Unit Testing with Mocked Dio

Use `dio`'s `DioAdapter` (via `http_mock_adapter`) or mock the `Dio` instance with `mocktail`/`mockito`:

```dart
class MockDio extends Mock implements Dio {}

test('getUsers returns list of users on success', () async {
  final dio = MockDio();
  when(() => dio.get('/users')).thenAnswer(
    (_) async => Response(
      requestOptions: RequestOptions(path: '/users'),
      data: {'data': [/* mock json */]},
      statusCode: 200,
    ),
  );

  final result = await UserRemoteDataSource(dio).getUsers();
  expect(result, isA<List<User>>());
});
```

### 13.2 Rules

- Every data-source method must have a corresponding unit test covering: success, timeout, bad response, and no-internet cases.
- Do not hit real network endpoints in unit/widget tests.
- Use integration tests (`integration_test` package) sparingly against a staging environment for critical flows only.

---

## 14. Retry & Resilience

Use `dio_smart_retry` or a custom `RetryInterceptor` for idempotent GET requests only:

```dart
dio.interceptors.add(
  RetryInterceptor(
    dio: dio,
    retries: 3,
    retryDelays: const [
      Duration(seconds: 1),
      Duration(seconds: 2),
      Duration(seconds: 3),
    ],
  ),
);
```

**Rule:** Never auto-retry non-idempotent requests (POST/PATCH/DELETE) without explicit idempotency-key support from the backend.

---

## 15. Code Review Checklist

Before merging any PR touching networking code, verify:

- [ ] No `Dio()` instantiated outside `DioClient`
- [ ] Timeouts explicitly set
- [ ] Errors caught and mapped to `ApiException`
- [ ] No sensitive data logged
- [ ] CancelToken used where request is tied to widget lifecycle
- [ ] Unit tests cover success + failure paths
- [ ] No hardcoded URLs/tokens in code
- [ ] Base URL driven by environment config

---

## 16. Common Pitfalls to Avoid

| Pitfall | Correct Approach |
|---|---|
| Creating a new `Dio()` per API call | Use a singleton/injected instance |
| Catching generic `Exception` instead of `DioException` | Explicitly catch `on DioException` |
| Logging full response bodies in production | Gate logging behind `kDebugMode` |
| No timeout set (defaults to indefinite wait) | Always set connect/receive/send timeouts |
| Parsing JSON directly in UI widgets | Keep parsing in data layer/models |
| Ignoring `CancelToken` for search/autocomplete | Cancel superseded requests |
| Retrying POST requests blindly | Restrict auto-retry to idempotent GETs |

---

## 17. References

- Official Dio package: https://pub.dev/packages/dio
- Dio GitHub repository and docs: https://github.com/cfug/dio

---

**Revision History**

| Version | Date | Change |
|---|---|---|
| 1.0 | Initial release | Baseline SOP for Dio integration |
