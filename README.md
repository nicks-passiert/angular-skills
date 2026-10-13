# Angular Skills

Four [agent skills](https://code.claude.com/docs/en/skills) that hold Claude to a specific way of
writing Angular. Each one is a rule with a review procedure: point it at a file pattern and it
reports pass/fail per component, or let it load on its own while components are written.

| Skill | Rule | Angular |
| ----- | ---- | ------- |
| [`angular-host-as-wrapper`](skills/angular-host-as-wrapper/SKILL.md) | The host element is the wrapper. No inner `<div>` that only carries layout classes — they go in `host: { class }`. | any |
| [`angular-onpush`](skills/angular-onpush/SKILL.md) | Every component runs OnPush: written out before v22, omitted from v22 where it is the default. Rejects the `ChangeDetectionStrategy.Eager` line the v22 migration inserts. | any |
| [`angular-signal-forms`](skills/angular-signal-forms/SKILL.md) | Forms use `form()`, `FormField` and `[formField]` from `@angular/forms/signals` — not `ngModel`, `model()` or `FormGroup`/`FormControl`. | 22+ |
| [`angular-signal-forms-date`](skills/angular-signal-forms-date/SKILL.md) | Date inputs are `Date \| null` fields constrained with `minDate()`/`maxDate()`, with date-only UTC values. | 22+ |

The code in the two forms skills compiles against Angular 22.1 with `strictTemplates`.

## How these relate to the official Angular skills

The Angular team publishes [`angular/skills`](https://github.com/angular/skills), whose
`angular-developer` skill is the reference for the framework's APIs. Install that first. These
skills sit on top of it and differ in two ways:

- **They enforce, where the official skill describes.** `angular-developer` explains host bindings;
  `angular-host-as-wrapper` fails a component that wraps its template in a layout `<div>`.
- **They are stricter.** The official guidance keeps an existing form on whatever approach it
  already uses. `angular-signal-forms` flags `ngModel` and reactive forms as violations in any
  Angular 22+ project. Skip that skill if you are not migrating.

`angular-signal-forms-date` covers ground the official reference does not: `minDate()`/`maxDate()`,
`Date`-typed fields, and the UTC off-by-one that comes with them.

## Install

**Claude Code, as a plugin:**

```
/plugin marketplace add nicks-passiert/angular-skills
/plugin install angular-skills@nicks-passiert
```

**Any agent that reads skills, through [skills.sh](https://skills.sh/):**

```bash
npx skills add nicks-passiert/angular-skills
```

**By hand:** copy the folders you want from `skills/` into `~/.claude/skills/` (all projects) or
`.claude/skills/` (one project).

## Use

The skills load on their own when a component or form is created or changed. To review existing
code, call one with a file pattern:

```
/angular-host-as-wrapper libs/ui/**/*.ts
/angular-onpush
/angular-signal-forms apps/admin/**/*.ts
```

Installed as a plugin, the names carry the plugin prefix: `/angular-skills:angular-onpush`.

## License

[MIT](LICENSE)
