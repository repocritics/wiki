# jaredpalmer/formik

> React form state library that centralizes values, validation, and submission — the incumbent that popularized "forms without the tears," now in low-maintenance mode.

[GitHub repo](https://github.com/jaredpalmer/formik) ·
[Official website](https://formik.org) ·
[License: Apache-2.0](https://github.com/jaredpalmer/formik/blob/main/LICENSE)

## Overview

Formik is a React library for managing form state: it holds field values, touched/error metadata, and submission status in one place, and exposes them through a component (`<Formik>`), a hook (`useFormik`), and helper components (`<Field>`, `<Form>`, `<ErrorMessage>`). It was created by Jared Palmer in 2017 and became the default answer to "how do I do forms in React" for several years[^1]. Validation is pluggable but most codebases pair it with Yup schemas via the `validationSchema` prop.

The library's defining tension is its state model. Formik keeps all form state in a single React state object and re-renders subscribing components on every change. This is simple to reason about and made Formik easy to learn, but it means a keystroke in one field can re-render the entire form. On small-to-medium forms this is invisible; on large forms with many fields or expensive children it becomes the dominant performance problem, and it is the reason a large share of the ecosystem migrated to uncontrolled alternatives.

The second thing to know about Formik in 2026 is maintenance cadence. The repository has ~34k stars and heavy historical usage, but development has slowed markedly since the original author moved on to other work[^2]. Hundreds of open issues and long gaps between releases are the current reality. Formik is best understood as a stable, widely-deployed, feature-frozen library rather than an actively evolving one — a fine default for existing code, a weaker default for greenfield projects.

## Getting Started

```bash
npm install formik yup
```

```tsx
import { useFormik } from "formik";
import * as Yup from "yup";

const schema = Yup.object({
  email: Yup.string().email("Invalid email").required("Required"),
  password: Yup.string().min(8, "Too short").required("Required"),
});

export function SignupForm() {
  const formik = useFormik({
    initialValues: { email: "", password: "" },
    validationSchema: schema,
    onSubmit: (values) => {
      // values is fully typed from initialValues
      console.log(values);
    },
  });

  return (
    <form onSubmit={formik.handleSubmit}>
      <input
        name="email"
        onChange={formik.handleChange}
        onBlur={formik.handleBlur}
        value={formik.values.email}
      />
      {formik.touched.email && formik.errors.email ? (
        <div>{formik.errors.email}</div>
      ) : null}
      <button type="submit">Submit</button>
    </form>
  );
}
```

The `<Formik>` render-props form and the `useFormik` hook are equivalent surfaces over the same core; the hook is the recommended path since v2.

## Architecture / How It Works

Formik centralizes state in a reducer-backed store held by the `<Formik>` provider (or created by `useFormik`). Three parallel objects track the form: `values`, `errors`, and `touched`, each keyed by field name (dot/bracket paths for nested and array fields). `handleChange` reads the DOM event's `name` and `value` and writes back into `values`; it relies on inputs being named correctly, which is a common source of silent bugs when `name` is missing or wrong.

Field wiring happens two ways. `<Field name="x">` and the `useField("x")` hook subscribe to a slice of the store via React context; `<FastField>` exists specifically to opt a field out of re-rendering unless its own slice or shared config changes, and is the library's built-in escape hatch for large forms[^3]. `<FieldArray>` provides helpers (`push`, `remove`, `swap`, `move`) for dynamic lists.

Validation runs through one of two props: `validate` (a function returning an errors object) or `validationSchema` (a Yup — or Yup-compatible — schema). Formik runs validation on change and blur by default (`validateOnChange`, `validateOnBlur`), so on a large form each keystroke can trigger a full schema validation in addition to the re-render. Submission is gated: `handleSubmit` marks all fields touched, runs validation, and only calls `onSubmit` if there are no errors, threading `isSubmitting` and `setSubmitting` through so the UI can disable controls.

The core coupling to understand: everything hangs off one context value, and React re-renders context consumers when that value changes. Formik mitigates this with `<FastField>` and bailout logic, but the model is fundamentally controlled/subscription-based. This is the architectural fork in the road versus React Hook Form, which registers uncontrolled inputs by ref and re-renders as little as possible.

## Production Notes

- **Re-render cost scales with field count.** Forms with dozens of fields, or fields whose children are expensive (rich editors, large select lists), will feel input lag because a change anywhere re-renders subscribers. `<FastField>` is the intended fix, but it changes semantics (a `<FastField>` does not re-render when *other* fields change), so cross-field-dependent UI needs care or a manual `shouldUpdate`.
- **Yup is effectively a peer dependency in practice.** It is not required, but the documented happy path assumes it, and Yup adds meaningful bundle weight. For size-sensitive apps consider a lighter resolver or plain `validate` functions.
- **Nested/array paths are string-based.** `values.address.street` is addressed as `"address.street"`; typos in these strings are not caught by TypeScript and fail silently. Typing improves with `useField`, but path strings remain a footgun.
- **Maintenance risk is the real production concern.** With slow release cadence and a large open-issue backlog, do not expect timely fixes for edge cases, React version bumps, or React 19/Server Components interplay. Teams standardizing on Formik today should treat it as frozen infrastructure they may end up patching themselves[^2].
- **Performance advice from the docs is load-bearing.** The official "optimization" guidance (use `<FastField>`, avoid inline objects/functions in render, memoize) is not optional polish on large forms — it is the difference between usable and unusable.

## When to Use / When Not

**Use when:**
- You already have Formik across a codebase and migration cost outweighs the benefits of switching.
- Your forms are small-to-medium and you want a well-documented, widely-understood API with abundant Stack Overflow coverage.
- You want the render-props/`<Field>` component ergonomics specifically.

**Avoid when:**
- You're starting greenfield and care about form performance or bundle size — React Hook Form is the more common modern default.
- You have large or deeply dynamic forms where per-keystroke re-renders will hurt.
- You need a library under active development with fast response to React version changes.

## Alternatives

- react-hook-form/react-hook-form — use instead for greenfield work; uncontrolled/ref-based, fewer re-renders, smaller bundle, actively maintained.
- TanStack/form — use when you want a type-first, framework-agnostic form core with first-class TypeScript inference.
- final-form/react-final-form — use when you want a subscription-based model (opt into exactly the state slices you need) from Formik's own conceptual lineage.
- react-hook-form/resolvers + colinhacks/zod — use when you want Zod schema validation; pairs with React Hook Form rather than Formik.
- unform/unform — use for uncontrolled React Native-friendly forms (less active, niche).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9 | 2017 | Initial public releases; render-props `<Formik>` API[^1]. |
| 1.0 | 2018-08 | First stable major; TypeScript-first, `<Field>`/`<FieldArray>` matured. |
| 2.0 | 2019-10 | Hooks: `useFormik`, `useField`, `useFormikContext`; `<FastField>` for perf[^4]. |
| 2.2 | 2020 | Incremental fixes and type improvements. |
| 2.4 | 2023–2024 | Maintenance releases; cadence slows notably[^2]. |

## References

[^1]: Formik documentation and project site. https://formik.org
[^2]: Formik issue tracker and release history — open-issue backlog and release cadence. https://github.com/jaredpalmer/formik/issues
[^3]: Formik docs, "FastField" — opt-out re-render optimization. https://formik.org/docs/api/fastfield
[^4]: Formik docs, "useFormik" and hooks API introduced in v2. https://formik.org/docs/api/useFormik

## Tags

react, forms, form-validation, typescript, state-management, hooks, yup, react-native, frontend, form-state
