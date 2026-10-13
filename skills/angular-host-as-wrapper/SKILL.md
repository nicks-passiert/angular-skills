---
name: angular-host-as-wrapper
description: Enforce that Angular components use their host element as the wrapper (not an inner wrapping div), with layout classes via host { class } in @Component metadata. Use when reviewing existing components for compliance or when creating/modifying components.
argument-hint: '[file-pattern]'
---

# Angular Component Host as Wrapper

Components MUST use their **host element** (the component's selector tag) as the wrapping container. Never add a wrapping `<div>` inside the template just to apply layout or positioning classes.

Use `$ARGUMENTS` as the file pattern when reviewing (default: `**/*.ts` files containing `@Component(`, excluding `node_modules`).

## Correct pattern

```typescript
@Component({
  selector: 'app-forgot-password',
  template: `
    <form (submit)="onSubmit($event)" class="w-full max-w-md">
      <!-- content -->
    </form>
  `,
  host: {
    class: 'flex flex-auto justify-center',
  },
})
```

The component's selector tag (`<app-forgot-password>`) becomes the layout container. Positioning classes (`flex flex-auto justify-center`) are applied to the host via `host: { class: '...' }`. The template starts directly with meaningful content (the `<form>`).

The examples use Tailwind utility classes; the rule is the same with any class names.

## Wrong pattern

```typescript
@Component({
  selector: 'app-forgot-password',
  template: `
    <div class="flex flex-auto justify-center">
      <form (submit)="onSubmit($event)" class="w-full max-w-md">
        <!-- content -->
      </form>
    </div>
  `,
})
```

The wrapping `<div>` is redundant — the host element is already there. Remove the `<div>`, move its classes to `host: { class: '...' }`. Same rendered layout, one fewer DOM node.

## Rules

1. **Host element IS the wrapper.** The component's selector tag is the layout container — don't add a `<div>` (or other non-semantic element) inside the template that just wraps everything.
2. **Apply layout/positioning classes via `host: { class: '...' }`** in the `@Component` metadata.
3. **Template starts with meaningful content** — the first element should be the actual content (a `<form>`, `<header>`, `<section>`, etc.), not a generic `<div>`.
4. **Give the host a display value.** A custom element is `display: inline` by default, so a wrapper `<div>` that was a plain block needs `block` (or `flex`, `grid`, `contents`) among the host classes when its classes move over — otherwise width, padding and margin stop behaving as they did.

## When reviewing

For each component file:

1. Read the file.
2. Check whether the template starts with a `<div>` (or similar non-semantic wrapper) containing the entire rest of the template.
3. If yes — and the wrapper exists only to apply classes — flag as a violation. Fix: remove the wrapper, move its classes to `host: { class: '...' }`.

### Report format

| File:line | Component | Status    | Issue       |
| --------- | --------- | --------- | ----------- |
| path:line | Name      | pass/fail | Description |

## Exceptions

- **Multiple top-level children** — if the template genuinely has multiple siblings at the root (e.g. `<header>` + `<main>` + `<footer>`), the host alone can't group them and a wrapper is justified. Prefer a semantic element (`<article>`, `<section>`) over a `<div>` when possible.
- **Dynamic structural elements** — `@if` / `@for` at the template root are conditional wrappers; this is fine.

## When creating or modifying components

- Default: no wrapping `<div>` at the top of the template.
- Default: apply all layout/positioning classes via `host: { class: '...' }`.
- Reach for a wrapper element only when the template genuinely needs one (multiple top-level children, semantic grouping).

## Why this matters

- **Saves a DOM node** per component instance — multiplied by every render.
- **Cleaner CSS targeting** — the component selector itself (`app-forgot-password`) is a CSS-targetable element; styling the host is more semantic than styling a class on an internal div.
- **Ties styling to component identity** — host classes describe how the component places itself in its parent layout, which is the component's responsibility.
