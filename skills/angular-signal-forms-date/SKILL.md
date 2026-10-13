---
name: angular-signal-forms-date
description: Apply min/max constraints to date inputs used with Angular signal forms — Date-typed fields with minDate()/maxDate(), bound via [formField] (Angular 22+). Use when a form has an <input type="date">, a date range, or an NG8022 error about min/max on a [formField] element.
---

# Date Field Constraints with Signal Forms

Angular's `FormField` directive from `@angular/forms/signals` (named `Field` with selector `[field]` before Angular 22; `Field` is now a type alias for the field-tree accessor) blocks all direct bindings (`[min]`, `[attr.min]`, `min="{{expr}}"`) on its host element — constraint metadata must flow through the form schema. How to express a date constraint depends on the **type of the form-model field**.

## Preferred (Angular 22+): model dates as `Date | null` + `minDate()` / `maxDate()`

Angular 22 added `minDate()` / `maxDate()` schema validators (`@publicApi 22.0`). Their path type is
`SchemaPath<TValue extends Date | null>`, so the form model must store a **`Date` object**, not a
date string. For such fields, `FormField` does everything itself (verified in the v22.1 source):

- binds `<input type="date">` via `valueAsDate` — no string conversion in the component
- syncs `minDate()`/`maxDate()` metadata to the **native `min`/`max` DOM properties**
  (formatted `YYYY-MM-DD`), so the browser picker blocks out-of-range dates
- reports `minDate`/`maxDate` validation errors

Cross-field constraints use the validators' `LogicFn` parameter with `ctx.valueOf()` — no
`computed()` needed:

```typescript
private readonly today = utcToday(); // declare BEFORE the form — the schema closure captures it

readonly model = signal<{ startsAt: Date | null; endsAt: Date | null }>({
  startsAt: utcToday(),
  endsAt: null,
});

readonly myForm = form(this.model, (m) => {
  required(m.startsAt);
  minDate(m.startsAt, this.today);
  minDate(m.endsAt, (ctx) => ctx.valueOf(m.startsAt) ?? this.today);
});
```

```html
<input type="date" [formField]="myForm.startsAt" />
```

**UTC caveat:** the min/max formatter uses UTC getters (`getUTCFullYear` etc.), so construct
constraint dates as **date-only UTC** values to avoid off-by-one-day errors in non-UTC timezones.
Serialize at the submit boundary with `.toISOString()` — for date-only UTC Dates this yields a
midnight-UTC ISO timestamp.

Two small helpers cover both directions:

```typescript
/** Today as a date-only UTC value (midnight UTC of the current UTC day). */
export function utcToday(): Date {
  return new Date(new Date().toISOString().slice(0, 10));
}

/** An ISO date or timestamp from the backend as a date-only UTC value. */
export function isoToUtcDate(iso: string): Date {
  return new Date(iso.slice(0, 10));
}
```

## Fallback: custom directive for string-typed date fields

Only when the form model must keep the date as a **string** (legacy models, no migration budget):
`minDate()` won't type-check, and the built-in `min()`/`max()` only accept numbers. Then a small
directive that sets the native attribute via `ElementRef` is still the correct workaround — it
coexists with `[formField]` without conflict.

```typescript
import { Directive, effect, ElementRef, inject, input } from '@angular/core';

@Directive({
  selector: 'input[type=date][appDateMin]', // replace `app` with your project's prefix
})
export class DateMinDirective {
  readonly appDateMin = input.required<string>();

  private readonly el = inject<ElementRef<HTMLInputElement>>(ElementRef);

  constructor() {
    effect(() => {
      this.el.nativeElement.min = this.appDateMin();
    });
  }
}
```

For `dateMax`, follow the same pattern with `this.el.nativeElement.max`. Dynamic constraints derive
the min string via `computed()` in the component and bind `[appDateMin]="endDateMin()"`.

## Why not the built-in `min()` / `max()` validators?

Their `minValue` parameter only accepts `number` — not a date string like `"2026-03-11"`. They are
designed for numeric inputs; dates get the dedicated `minDate()`/`maxDate()` (Angular 22+).

## Why not `[attr.min]` or `[min]`?

`FormField` maintains a banned-bindings list: `min`, `max`, `minLength`, `maxLength`, `required`, `disabled`, `readonly`, etc. Any of these on a `[formField]` host element produces compiler error **NG8022**:

```
NG8022: Binding to '[attr.min]' is not allowed on nodes using the '[formField]' directive
```

This is by design — constraint metadata flows through the form schema, and since Angular 22 the schema covers dates via `minDate()`/`maxDate()`. The directive fallback exists only because string-typed fields fall outside that coverage.

## Rules

1. **Model date fields as `Date | null`** and constrain them with `minDate()` / `maxDate()` in the schema (Angular 22+). Reach for the directive fallback only when the model is stuck on strings.
2. **Never use** `[min]`, `[attr.min]`, or `min="{{expr}}"` on a `[formField]` element — fails compilation (NG8022).
3. **Cross-field constraints** go through the validator's `LogicFn` with `ctx.valueOf()` — not `computed()`.
4. **Constraint dates must be date-only UTC** values; serialize with `.toISOString()` at the submit boundary.
5. If the fallback directive is used, its **selector needs a project prefix** (e.g. `app`) to satisfy `@angular-eslint/directive-selector`.
