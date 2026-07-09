# Flutter Theme Usage SOP (Material 3)

This SOP defines how to consume Flutter's `ThemeData` and `ColorScheme` consistently across the app. Following it keeps the UI visually consistent, accessible, and automatically compatible with light and dark mode — with zero manual dark-mode patching.

---

## 0. Setup — Get These Once, Reuse Everywhere

At the top of any widget's `build()` method:

```dart
final theme = Theme.of(context);
final colors = theme.colorScheme;
final textTheme = theme.textTheme;
```

Everything below refers to `colors.xxx` and `textTheme.xxx` from this setup.

---

## 1. Golden Rule

**❌ Never** use raw colors inside widgets:

```dart
Colors.blue
Colors.green
Colors.grey
Colors.white
Colors.black
```

**✅ Always** use a semantic role from `colors` or `textTheme` instead. The only exception is a genuinely fixed brand mark (e.g. a logo) or a status color not modeled by Material (see §16).

> **Why:** Raw colors don't adapt to dark mode, don't respond to dynamic/brand theming, and break contrast guarantees. Semantic roles do all three automatically.

---

## 2. Quick Reference Table

| Element | Property | Role |
|---|---|---|
| Screen background | `backgroundColor` | `colors.surface` |
| AppBar background | `backgroundColor` | `colors.surface` |
| AppBar title/icons | `foregroundColor` | `colors.onSurface` |
| NavigationBar background | `backgroundColor` | `colors.surface` |
| NavigationBar selected item | — | `colors.primary` |
| NavigationBar indicator | — | `colors.primaryContainer` |
| Card (default) | `color` | `colors.surfaceContainer` |
| Card (low emphasis) | `color` | `colors.surfaceContainerLow` |
| Card (elevated) | `color` | `colors.surfaceContainerHigh` |
| Primary button background | — | `colors.primary` |
| Primary button text/icon | — | `colors.onPrimary` |
| Filled tonal button background | — | `colors.secondaryContainer` |
| Filled tonal button text | — | `colors.onSecondaryContainer` |
| Outlined button border | — | `colors.outline` |
| Outlined/Text button label | — | `colors.primary` |
| Heading / body text | `color` | `colors.onSurface` |
| Subtitle / hint text | `color` | `colors.onSurfaceVariant` |
| Selected icon | `color` | `colors.primary` |
| Default icon | `color` | `colors.onSurfaceVariant` |
| Disabled icon | `color` | `colors.outline` |
| Input field fill | `fillColor` | `colors.surfaceContainerHighest` |
| Input field border | `borderSide` | `colors.outline` |
| Input field focused border | `borderSide` | `colors.primary` |
| Input hint text | `color` | `colors.onSurfaceVariant` |
| Divider | `color` | `colors.outlineVariant` |
| Chip background | `backgroundColor` | `colors.secondaryContainer` |
| Chip text | `labelStyle.color` | `colors.onSecondaryContainer` |
| Chip selected | `selectedColor` | `colors.primaryContainer` |
| Dialog background | `backgroundColor` | `colors.surfaceContainerHigh` |
| Dialog title | `color` | `colors.onSurface` |
| Dialog body | `color` | `colors.onSurfaceVariant` |
| Bottom sheet background | `backgroundColor` | `colors.surface` |
| FAB background | `backgroundColor` | `colors.primary` |
| FAB icon | `foregroundColor` | `colors.onPrimary` |
| Progress indicator (active) | `color` | `colors.primary` |
| Progress indicator (track) | `backgroundColor` | `colors.surfaceContainerHighest` |

Use this table as the fast lookup; the sections below give the same information with code examples and rationale.

---

## 3. Screen Background

```dart
Scaffold(
  backgroundColor: colors.surface,
)
```

For a subtly recessed section within a screen, use `colors.surfaceContainerLowest` instead of `colors.surface`.

---

## 4. AppBar

```dart
AppBar(
  backgroundColor: colors.surface,
  foregroundColor: colors.onSurface,
)
```

`foregroundColor` covers the title, back button, and default action icon tint in one line.

---

## 5. NavigationBar

```dart
NavigationBar(
  backgroundColor: colors.surface,
  indicatorColor: colors.primaryContainer,
  destinations: [...],
)
```

Selected label/icon color is `colors.primary`; unselected typically falls back to `colors.onSurfaceVariant`.

---

## 6. Cards

| Emphasis | Role |
|---|---|
| Elevated / prominent | `colors.surfaceContainerHigh` |
| Normal | `colors.surfaceContainer` |
| Low emphasis / nested | `colors.surfaceContainerLow` |

```dart
Card(
  color: colors.surfaceContainer,
  child: ...,
)
```

Use the surface-container scale to create visual hierarchy between stacked/nested cards without introducing new colors or manual elevation shadows.

---

## 7. Buttons

**Primary (`FilledButton` / `ElevatedButton`)**

```dart
FilledButton(
  style: FilledButton.styleFrom(
    backgroundColor: colors.primary,
    foregroundColor: colors.onPrimary,
  ),
  onPressed: () {},
  child: const Text('Continue'),
)
```

**Filled tonal (`FilledButton.tonal`)**

```dart
FilledButton.tonal(
  style: FilledButton.styleFrom(
    backgroundColor: colors.secondaryContainer,
    foregroundColor: colors.onSecondaryContainer,
  ),
  onPressed: () {},
  child: const Text('Save draft'),
)
```

**Outlined**

```dart
OutlinedButton(
  style: OutlinedButton.styleFrom(
    side: BorderSide(color: colors.outline),
    foregroundColor: colors.primary,
  ),
  onPressed: () {},
  child: const Text('Cancel'),
)
```

**Text button**

```dart
TextButton(
  style: TextButton.styleFrom(foregroundColor: colors.primary),
  onPressed: () {},
  child: const Text('Learn more'),
)
```

> These styles should be set **once** in the app's `ThemeData` (`filledButtonTheme`, `outlinedButtonTheme`, `textButtonTheme`) rather than repeated per call site. See §17.

---

## 8. Text

```dart
Text('Order confirmed', style: textTheme.titleLarge?.copyWith(color: colors.onSurface))
Text('Body copy goes here', style: textTheme.bodyMedium?.copyWith(color: colors.onSurface))
Text('Delivered 2 days ago', style: textTheme.bodySmall?.copyWith(color: colors.onSurfaceVariant))
```

| Text role | Color |
|---|---|
| Heading | `colors.onSurface` |
| Body | `colors.onSurface` |
| Subtitle / caption | `colors.onSurfaceVariant` |
| Hint / placeholder | `colors.onSurfaceVariant` |

Use `.copyWith()` only to adjust color/weight — never rebuild a `TextStyle` from scratch inside a widget.

---

## 9. Icons

```dart
Icon(Icons.check, color: colors.primary)       // selected
Icon(Icons.info_outline, color: colors.onSurfaceVariant) // default
Icon(Icons.block, color: colors.outline)       // disabled
```

---

## 10. Input Fields

```dart
TextField(
  decoration: InputDecoration(
    filled: true,
    fillColor: colors.surfaceContainerHighest,
    hintStyle: TextStyle(color: colors.onSurfaceVariant),
    enabledBorder: OutlineInputBorder(
      borderSide: BorderSide(color: colors.outline),
    ),
    focusedBorder: OutlineInputBorder(
      borderSide: BorderSide(color: colors.primary, width: 2),
    ),
  ),
  style: TextStyle(color: colors.onSurface),
)
```

As with buttons, define this once in `ThemeData.inputDecorationTheme` — see §17.

---

## 11. Dividers

```dart
Divider(color: colors.outlineVariant)
```

`outlineVariant` is intentionally lower-contrast than `outline`; it's meant for separators, not borders that need to be clearly visible.

---

## 12. Chips

```dart
Chip(
  backgroundColor: colors.secondaryContainer,
  labelStyle: TextStyle(color: colors.onSecondaryContainer),
)

ChoiceChip(
  selected: isSelected,
  selectedColor: colors.primaryContainer,
  label: const Text('Filter'),
)
```

---

## 13. Dialogs

```dart
AlertDialog(
  backgroundColor: colors.surfaceContainerHigh,
  title: Text('Delete item?', style: TextStyle(color: colors.onSurface)),
  content: Text('This can't be undone.', style: TextStyle(color: colors.onSurfaceVariant)),
)
```

---

## 14. Bottom Sheets

```dart
showModalBottomSheet(
  backgroundColor: colors.surface,
  context: context,
  builder: (_) => ...,
)
```

---

## 15. Floating Action Button

```dart
FloatingActionButton(
  backgroundColor: colors.primary,
  foregroundColor: colors.onPrimary,
  onPressed: () {},
  child: const Icon(Icons.add),
)
```

---

## 16. Progress Indicators

```dart
CircularProgressIndicator(
  color: colors.primary,
  backgroundColor: colors.surfaceContainerHighest,
)
```

---

## 17. Status Colors

### Error (built into Material 3 — always use these, never a custom red)

```dart
colors.error
colors.onError
colors.errorContainer
colors.onErrorContainer
```

Use for: invalid form fields, failed validation, destructive-action confirmation, network/request failures.

```dart
Text('Invalid email address', style: TextStyle(color: colors.error))
```

### Success, Warning, Info (not in Material's default `ColorScheme` — define via `ThemeExtension`)

Material 3 has no built-in success/warning/info roles, so hardcoding a green/amber/blue directly in widgets is one of the few acceptable exceptions to the Golden Rule — **but only if it goes through a single shared extension**, not scattered literals.

```dart
@immutable
class AppStatusColors extends ThemeExtension<AppStatusColors> {
  const AppStatusColors({
    required this.success,
    required this.onSuccess,
    required this.warning,
    required this.onWarning,
    required this.info,
    required this.onInfo,
  });

  final Color success, onSuccess;
  final Color warning, onWarning;
  final Color info, onInfo;

  static const light = AppStatusColors(
    success: Color(0xFF2E7D32), onSuccess: Colors.white,
    warning: Color(0xFFF59E0B), onWarning: Colors.black,
    info: Color(0xFF0288D1), onInfo: Colors.white,
  );

  static const dark = AppStatusColors(
    success: Color(0xFF66BB6A), onSuccess: Colors.black,
    warning: Color(0xFFFFB74D), onWarning: Colors.black,
    info: Color(0xFF4FC3F7), onInfo: Colors.black,
  );

  @override
  AppStatusColors copyWith({Color? success, Color? onSuccess, Color? warning, Color? onWarning, Color? info, Color? onInfo}) {
    return AppStatusColors(
      success: success ?? this.success,
      onSuccess: onSuccess ?? this.onSuccess,
      warning: warning ?? this.warning,
      onWarning: onWarning ?? this.onWarning,
      info: info ?? this.info,
      onInfo: onInfo ?? this.onInfo,
    );
  }

  @override
  AppStatusColors lerp(ThemeExtension<AppStatusColors>? other, double t) {
    if (other is! AppStatusColors) return this;
    return AppStatusColors(
      success: Color.lerp(success, other.success, t)!,
      onSuccess: Color.lerp(onSuccess, other.onSuccess, t)!,
      warning: Color.lerp(warning, other.warning, t)!,
      onWarning: Color.lerp(onWarning, other.onWarning, t)!,
      info: Color.lerp(info, other.info, t)!,
      onInfo: Color.lerp(onInfo, other.onInfo, t)!,
    );
  }
}
```

Register it in `ThemeData(extensions: [AppStatusColors.light])` (and `.dark` for the dark theme), then consume it the same way as `colorScheme`:

```dart
final status = Theme.of(context).extension<AppStatusColors>()!;

Icon(Icons.check_circle, color: status.success);
Container(color: status.warning, child: Text('Pending', style: TextStyle(color: status.onWarning)));
```

| Status | Use for |
|---|---|
| Success | Confirmed order, correct answer, completed action |
| Warning | Approaching limit, unsaved changes, non-blocking issue |
| Info | Tips, neutral announcements, informational banners |

---

## 18. Wire Component Styles Into `ThemeData` Once

Don't repeat `FilledButton.styleFrom(...)` or `InputDecoration(...)` at every call site. Define them once:

```dart
ThemeData(
  useMaterial3: true,
  colorScheme: colorScheme,
  filledButtonTheme: FilledButtonThemeData(
    style: FilledButton.styleFrom(
      backgroundColor: colorScheme.primary,
      foregroundColor: colorScheme.onPrimary,
    ),
  ),
  outlinedButtonTheme: OutlinedButtonThemeData(
    style: OutlinedButton.styleFrom(
      side: BorderSide(color: colorScheme.outline),
      foregroundColor: colorScheme.primary,
    ),
  ),
  inputDecorationTheme: InputDecorationTheme(
    filled: true,
    fillColor: colorScheme.surfaceContainerHighest,
    enabledBorder: OutlineInputBorder(borderSide: BorderSide(color: colorScheme.outline)),
    focusedBorder: OutlineInputBorder(borderSide: BorderSide(color: colorScheme.primary, width: 2)),
  ),
  extensions: [AppStatusColors.light],
)
```

With this in place, widgets rarely need to pass `style:` at all — plain `FilledButton(child: ...)` already looks correct.

---

## 19. Accessibility Notes

- Every `onX` role (`onPrimary`, `onSurface`, `onSecondaryContainer`, etc.) is guaranteed by Material 3 to meet contrast requirements against its paired `X` role — don't mix roles from different pairs (e.g. don't put `onPrimary` text on a `secondaryContainer` background).
- When adding custom status colors (§17), verify contrast of `onSuccess`/`onWarning`/`onInfo` against their container manually (aim for WCAG AA, 4.5:1 for body text).
- Don't rely on color alone to convey status — pair with an icon or label (e.g. error text + error icon), since color-blind users may not distinguish error/success/warning by hue alone.

---

## 20. Code Review Checklist

- [ ] No `Colors.*` or raw `Color(0x...)` literals outside `theme/` (exception: registered status colors in `AppStatusColors`)
- [ ] All button/input/card styling comes from `ThemeData`, not inline `styleFrom()` per widget
- [ ] Text always styled via `textTheme.*`, colored via `colors.onSurface` / `colors.onSurfaceVariant`
- [ ] Screen tested in both light and dark mode
- [ ] Status colors (success/warning/info) only referenced via `AppStatusColors`, never inline hex
- [ ] Paired roles used correctly (e.g. `onPrimary` only on `primary`, never on `secondaryContainer`)

---

## 21. Summary

Material 3's `ColorScheme` gives you a complete, accessible, dark-mode-ready palette out of the box. The only engineering discipline required is: **always reach for a semantic role, never a raw color** — and for the handful of roles Material doesn't provide (success/warning/info), define them once as a `ThemeExtension` and treat them with the same discipline as everything else.
