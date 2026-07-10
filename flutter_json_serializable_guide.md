# Flutter JSON Serializable — Step-by-Step Guide

`json_serializable` is the recommended way to convert JSON into Dart objects and vice versa. It automatically generates the parsing code for you.

Instead of writing:

```dart
User.fromJson(Map<String, dynamic> json) {
  id = json['id'];
  name = json['name'];
}
```

the generator writes it for you.

## Step 1: Add Dependencies

In `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter

  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.5.0
  json_serializable: ^6.9.0
```

Run:

```bash
flutter pub get
```

## Step 2: Create Model

Example API response:

```json
{
  "id": 1,
  "name": "Laptop",
  "price": 50000
}
```

Create `lib/models/product.dart`:

```dart
import 'package:json_annotation/json_annotation.dart';

part 'product.g.dart';

@JsonSerializable()
class Product {
  final int id;
  final String name;
  final double price;

  Product({
    required this.id,
    required this.name,
    required this.price,
  });

  factory Product.fromJson(Map<String, dynamic> json)
      => _$ProductFromJson(json);

  Map<String, dynamic> toJson()
      => _$ProductToJson(this);
}
```

## Step 3: What is `part`?

```dart
part 'product.g.dart';
```

You do not create this file manually — `build_runner` generates it.

## Step 4: Generate Code

Run:

```bash
flutter pub run build_runner build
```

or (new command):

```bash
dart run build_runner build
```

Flutter generates `product.g.dart`. Example generated code:

```dart
Product _$ProductFromJson(Map<String, dynamic> json) => Product(
      id: json['id'] as int,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
    );

Map<String, dynamic> _$ProductToJson(Product instance) =>
    <String, dynamic>{
      'id': instance.id,
      'name': instance.name,
      'price': instance.price,
    };
```

You never edit this file.

## Step 5: Using the Model

**Convert JSON → Object**

```dart
final json = {
  "id": 1,
  "name": "Laptop",
  "price": 50000
};

final product = Product.fromJson(json);

print(product.name);
```

Output:

```
Laptop
```

**Convert Object → JSON**

```dart
final product = Product(
  id: 1,
  name: "Laptop",
  price: 50000,
);

final json = product.toJson();

print(json);
```

Output:

```json
{
  "id": 1,
  "name": "Laptop",
  "price": 50000
}
```

## Step 6: Nested Objects

API:

```json
{
  "id": 1,
  "name": "Laptop",
  "category": {
      "id": 10,
      "name": "Electronics"
  }
}
```

**Category Model**

```dart
import 'package:json_annotation/json_annotation.dart';

part 'category.g.dart';

@JsonSerializable()
class Category {
  final int id;
  final String name;

  Category({
    required this.id,
    required this.name,
  });

  factory Category.fromJson(Map<String, dynamic> json)
      => _$CategoryFromJson(json);

  Map<String, dynamic> toJson()
      => _$CategoryToJson(this);
}
```

**Product**

```dart
import 'package:json_annotation/json_annotation.dart';

part 'product.g.dart';

@JsonSerializable()
class Product {
  final int id;
  final String name;
  final Category category;

  Product({
    required this.id,
    required this.name,
    required this.category,
  });

  factory Product.fromJson(Map<String, dynamic> json)
      => _$ProductFromJson(json);

  Map<String, dynamic> toJson()
      => _$ProductToJson(this);
}
```

Generated code automatically calls `Category.fromJson(...)`.

## Step 7: List of Objects

API:

```json
[
  {
    "id": 1,
    "name": "Laptop"
  },
  {
    "id": 2,
    "name": "Mobile"
  }
]
```

Convert:

```dart
final products = (jsonData as List)
    .map((e) => Product.fromJson(e))
    .toList();
```

## Step 8: Rename JSON Keys

Suppose the API sends:

```json
{
    "product_name": "Laptop"
}
```

Dart field:

```dart
@JsonKey(name: "product_name")
final String name;
```

Generated code handles the mapping automatically.

## Step 9: Default Values

```dart
@JsonKey(defaultValue: 0)
final int quantity;
```

If the API omits `quantity`, it becomes `0`.

## Step 10: Ignore a Field

```dart
@JsonKey(includeFromJson: false, includeToJson: false)
final String localCache;
```

Useful for local-only fields that should not be serialized.

## Step 11: Nullable Fields

API:

```json
{
   "description": null
}
```

Model:

```dart
final String? description;
```

## Step 12: DateTime

API:

```json
{
   "created_at": "2026-07-10T12:30:00Z"
}
```

Model:

```dart
@JsonKey(name: "created_at")
final DateTime createdAt;
```

Generated code converts between ISO 8601 strings and `DateTime` automatically.

## Step 13: Enum

```dart
enum Status {
  active,
  inactive,
}
```

Model:

```dart
final Status status;
```

API:

```json
{
    "status": "active"
}
```

The generated code maps the string to the enum value (and back).

## Step 14: Regenerate After Changes

Whenever you modify your model, regenerate the code:

```bash
dart run build_runner build
```

Or keep it running while you develop:

```bash
dart run build_runner watch
```

If generated files become stale or conflicting, use:

```bash
dart run build_runner build --delete-conflicting-outputs
```
