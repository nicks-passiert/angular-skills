---
name: angular-onpush
description: Enforce OnPush change detection on Angular components — written out as ChangeDetectionStrategy.OnPush before Angular 22, omitted from v22 where it is the default. Use when reviewing existing components for compliance or when creating/modifying components.
argument-hint: '[file-pattern]'
---

# Angular OnPush Change Detection

Every Angular component runs with OnPush change detection. How that is expressed depends on the
Angular major, so read the `@angular/core` version from the project's `package.json` before
reviewing or writing a component:

| Angular | How OnPush is expressed                                                                                                  |
| ------- | ------------------------------------------------------------------------------------------------------------------------ |
| 22+     | It is the framework default: **omit `changeDetection`**. Only `ChangeDetectionStrategy.Eager` (the old `Default`) fails. |
| < 22    | Declare `changeDetection: ChangeDetectionStrategy.OnPush` and import `ChangeDetectionStrategy` from `@angular/core`.     |

The split exists because the two forms contradict each other: written out in a v22 project it is
noise the default already covers, and left out before v22 the component silently gets the eager
strategy.

Use `$ARGUMENTS` as the file pattern when reviewing (default: `**/*.ts` files containing
`@Component(`, excluding `node_modules`).

## The v22 migration shim

`ng update` to Angular 22 runs the `change-detection-eager` migration, which writes
`changeDetection: ChangeDetectionStrategy.Eager` into components that did not declare a strategy, to
preserve their old behaviour. Treat that line as a to-do, not as a decision: confirm the component
works under OnPush (state in signals, no reliance on unrelated events to refresh the view) and
remove it. angular-eslint's `prefer-on-push-component-change-detection` rule, part of its
recommended TypeScript config, reports it too.

## When reviewing existing components

For each component file:

1. Read the file.
2. **Angular 22+:** check that `@Component` metadata sets no eager strategy. `ChangeDetectionStrategy.Eager`
   or `.Default` fails — the fix is to remove the property, and the import if nothing else uses it.
   An explicit `OnPush` still behaves correctly: note it as redundant and remove it when the file
   is being edited anyway, rather than failing the component.
3. **Angular < 22:** check that the metadata includes `changeDetection: ChangeDetectionStrategy.OnPush`
   and that `ChangeDetectionStrategy` is imported from `@angular/core`.

### Report format

| File      | Component | Status    | Issue       |
| --------- | --------- | --------- | ----------- |
| path:line | Name      | pass/fail | Description |

Then list each failing component with file path and line number, what's missing, and the exact fix (add or remove the import and/or the metadata property).

## Examples

### Angular 22+

**Pass — default left alone:**

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `...`,
})
```

**Fail — eager strategy (what the v22 `change-detection-eager` migration inserts):**

```typescript
@Component({
  selector: 'app-example',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.Eager,
})
```

### Angular < 22

**Fail — missing OnPush:**

```typescript
@Component({
  selector: 'app-example',
  template: `...`,
})
```

**Fail — missing import:**

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush, // not imported
})
```

**Pass:**

```typescript
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
```

## When creating or modifying components

- **Angular 22+:** leave `changeDetection` out of the `@Component` decorator, and do not keep a
  `ChangeDetectionStrategy.Eager` line a migration added.
- **Angular < 22:** include `changeDetection: ChangeDetectionStrategy.OnPush` and import
  `ChangeDetectionStrategy` from `@angular/core`.
