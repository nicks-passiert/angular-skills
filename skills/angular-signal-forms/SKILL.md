---
name: angular-signal-forms
description: Enforce Angular signal-based form patterns using @angular/forms/signals (Angular 22+). Use when reviewing existing components or when creating/modifying components with form bindings to ensure form(), FormField, and [formField] are used instead of ngModel, model(), or reactive FormGroup/FormControl.
argument-hint: '[file-pattern]'
---

# Angular Signal-Based Forms

All form components MUST use Angular signal forms from `@angular/forms/signals`. Apply when reviewing existing components and when creating or modifying form components.

Use `$ARGUMENTS` as the file pattern when reviewing (default: `**/*.ts` files containing `@Component(`, excluding `node_modules`).

> **Version:** this skill targets **Angular 22+**, where signal forms are stable. Read the `@angular/forms` version from the project's `package.json` first. Angular 21 shipped the API as experimental with different names — the directive was `Field` bound with `[field]`; in 22 it is `FormField` with `[formField]`, and `Field` is a type alias for the field-tree accessor. Before 21 the API does not exist: leave such a project on its current forms approach and say so.

## Correct pattern

```typescript
import { Component, signal } from '@angular/core';
import { FormField, email, form, required, submit } from '@angular/forms/signals';

interface MyFormData {
  name: string;
  email: string;
}

@Component({
  selector: 'app-my-form',
  imports: [FormField],
  template: `
    <form (submit)="onSubmit($event)">
      <input [formField]="myForm.name" />
      <input [formField]="myForm.email" type="email" />
      <button type="submit">Save</button>
    </form>
  `,
})
export class MyComponent {
  readonly formModel = signal<MyFormData>({ name: '', email: '' });
  readonly myForm = form(this.formModel, (data) => {
    required(data.name);
    required(data.email);
    email(data.email);
  });

  onSubmit(event: Event) {
    event.preventDefault();
    submit(this.myForm, async () => {
      const values = this.formModel();
      // runs only when the form is valid — use values.name, values.email
    });
  }
}
```

### Key elements

1. **`<form>` wrapper** — form inputs must be inside a `<form>` with a `(submit)` handler (or `[formRoot]`, see below).
2. **`signal()` for the model** — typed interface holds the form data.
3. **`form()` for the field tree + validation** — wraps the model signal, accepts an optional validator callback.
4. **`FormField` directive** — imported in the component `imports` array.
5. **`[formField]="myForm.fieldName"`** — binds each input; supports nested access (`myForm.address.street`).
6. **Read values from the model signal** — `this.formModel()` returns current data.
7. **Read individual field state** — `this.myForm.fieldName().value()` for the value, `this.myForm().valid()` for overall validity.
8. **Update programmatically** — `this.formModel.set({...})` to replace the whole model, or `this.myForm.fieldName().value.set('new')` for a single field.

## Submitting

Submit through **`submit()`**. It marks every field as touched, so validation messages gated on `touched()` appear, and runs the action only when the form is valid. The action must be `async` (it returns a Promise).

A handwritten `if (this.myForm().valid()) { … }` guard still works, but it leaves untouched fields untouched — a user who presses Save on an empty form sees no errors. Replace it with `submit()` when the handler is edited.

`FormRoot` removes the handler altogether: `<form [formRoot]="myForm">` sets `novalidate`, prevents the default submit and calls `submit()` with the action given at form creation.

```typescript
import { FormField, FormRoot, form, required } from '@angular/forms/signals';

@Component({
  imports: [FormField, FormRoot],
  template: `
    <form [formRoot]="myForm">
      <input [formField]="myForm.name" />
      <button type="submit">Save</button>
    </form>
  `,
})
export class MyComponent {
  readonly formModel = signal({ name: '' });
  readonly myForm = form(
    this.formModel,
    (data) => {
      required(data.name);
    },
    {
      submission: {
        action: async () => {
          // runs only when the form is valid
        },
      },
    },
  );
}
```

## Import path

All signal form APIs come from `@angular/forms/signals`:

- `form` — creates the field tree from a model signal
- `FormField` — directive for `[formField]` template binding
- `FormRoot` — directive for `<form [formRoot]>`
- `submit` — validates, marks fields touched and runs the submit action
- Validators: `required`, `email`, `min`, `max`, `minLength`, `maxLength`, `pattern`, `minDate`, `maxDate`, and `validate` for custom rules

## Model structure

Define a TypeScript interface for type safety. Initialize all fields — never `undefined`:

```typescript
interface UserData {
  name: string;
  email: string;
  age: number | null; // an empty number input is null
  birthday: Date | null; // an empty date input is null
}

readonly model = signal<UserData>({ name: '', email: '', age: null, birthday: null });
```

- **Text fields are `string`, empty is `''`.** A `null` bound to a text input logs runtime warning NG01921 in dev mode and is coerced to `''`.
- **`null` is the empty value for number and date inputs** — `<input type="number">` and `<input type="date">` write `null` when cleared.
- Map `''` to `null` at the submit boundary when the backend wants a nullable string.

Nested models are supported:

```typescript
readonly model = signal({
  profile: { firstName: '', lastName: '' },
  address: { street: '', city: '', zip: '' },
});
readonly myForm = form(this.model);
// template: [formField]="myForm.address.city"
```

## Loading data asynchronously

Set the model signal — the field tree and UI update automatically:

```typescript
async loadData(id: string) {
  const data = await this.service.getData(id);
  this.formModel.set(data);
}
```

## When reviewing existing components

For each component file:

1. Read the file.
2. Check for the violations below.
3. Report findings.

### Violations to flag

#### 1. `[(ngModel)]` bindings (critical)

Never use `[(ngModel)]` or `FormsModule` for form fields. Use `[formField]` with signal forms instead.

**Fail:**
```typescript
import { FormsModule } from '@angular/forms';

@Component({
  imports: [FormsModule],
  template: `<input [(ngModel)]="name" />`,
})
export class BadComponent {
  name = '';
}
```

#### 2. `model()` used for form fields (critical)

`model()` is for component two-way binding inputs between parent/child, not for form fields. Form fields use `signal()` + `form()` + `[formField]`.

#### 3. Form inputs without a `<form>` wrapper (critical)

All `[formField]` inputs must be inside a `<form>` element with a `(submit)` handler or `[formRoot]`.

#### 4. Legacy `FormGroup` / `FormControl` reactive forms (critical)

**Fail:**
```typescript
import { FormGroup, FormControl, ReactiveFormsModule } from '@angular/forms';

readonly myForm = new FormGroup({
  name: new FormControl(''),
});
```

#### 5. Plain class properties bound to form inputs (critical)

Plain properties combined with OnPush will not trigger change detection when set asynchronously.

#### 6. Constraint attributes on a `[formField]` element (critical)

`min`, `max`, `minlength`, `maxlength`, `required`, `disabled`, `readonly`, `name` and `value` — static or bound — fail compilation with NG8022 next to `[formField]`. (A static `value` on a radio button or checkbox is the exception: it names the option.) Express them as rules in the schema (`min()`, `required()`, `disabled()`, `readonly()`, …).

### Report format

| File      | Component | Status    | Issue       |
| --------- | --------- | --------- | ----------- |
| path:line | Name      | pass/fail | Description |

Then list each failing component with file path, line number, what's wrong, and the fix using signal forms.

## When creating or modifying components

- Always define a TypeScript interface for the form data shape.
- Always use `signal()` for the model, `form()` for the field tree.
- Always import `FormField` from `@angular/forms/signals` and use `[formField]` bindings.
- Always wrap form inputs in a `<form>` with a `(submit)` handler or `[formRoot]`.
- With a `(submit)` handler, always call `event.preventDefault()`.
- Submit through `submit()` so fields are marked touched and the action runs only when valid.
- Use validators from `@angular/forms/signals`: `required`, `email`, `min`, `max`, `minLength`, `maxLength`, `pattern`, `minDate`, `maxDate`.
- Read submitted values from `this.formModel()`, not individual fields.
- Load async data with `this.formModel.set(data)` — the UI updates automatically.
