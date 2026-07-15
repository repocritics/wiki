# logaretm/vee-validate

> Form validation for Vue that handles the hard parts (state, errors, arrays, async) and leaves the markup to you.

[GitHub repo](https://github.com/logaretm/vee-validate) ·
[Official website](https://vee-validate.logaretm.com/v4) ·
[License: MIT](https://github.com/logaretm/vee-validate/blob/main/LICENSE)

## Overview

vee-validate is a form and validation library for Vue.js, authored and
maintained largely by Abdelrahman Awad (`logaretm`) since 2016[^1]. It is one of
the oldest surviving Vue form libraries and has been rewritten twice, each
rewrite tracking a shift in Vue's own API. It does not ship input widgets or
styling — it manages the state a form actually has (values, errors, touched/dirty
flags, submission state, field arrays) and lets you render whatever markup you
want. The tagline "Painless Vue forms" is accurate about its scope: it owns
validation and form state, nothing else.

The library's defining tension is API surface. It exposes two parallel styles —
a Composition API (`useForm`, `useField`, `defineField`) and higher-order
components (`Form`, `Field`, `ErrorMessage`) — that solve the same problem
differently, and most non-trivial apps end up mixing them[^2]. The Composition
API is the recommended path in current versions; the components are a thinner
convenience layer built on top of the same core. Validation itself is delegated:
you write a plain function, or hand it a Yup or Zod schema through the official
adapters. vee-validate deliberately does not own the schema language.

The major-version story matters when reading old tutorials. v2/v3 target Vue 2
and use a fundamentally different, template-directive / scoped-slot design; v4
and v5 target Vue 3 and are the composition-first rewrite[^3]. Code samples are
not portable across that line, and Stack Overflow answers from before ~2021 are
usually about a library that no longer exists in that form.

## Getting Started

```sh
npm install vee-validate
# schema adapters are separate, optional installs:
npm install @vee-validate/zod zod
```

```vue
<script setup>
import { useForm } from 'vee-validate';

const { defineField, handleSubmit, errors } = useForm({
  validationSchema: {
    email: (value) => (value?.includes('@') ? true : 'Enter a valid email'),
  },
});

const [email, emailProps] = defineField('email');

const onSubmit = handleSubmit((values) => {
  console.log(values); // validated payload
});
</script>

<template>
  <form @submit="onSubmit">
    <input v-model="email" v-bind="emailProps" />
    <span>{{ errors.email }}</span>
    <button>Submit</button>
  </form>
</template>
```

`defineField` returns a `v-model`-able ref plus a props bag that wires
blur/change events for validation timing. Swap the inline function for
`toTypedSchema(zodSchema)` to get validation and inferred value types from one
source.

## Architecture / How It Works

The core primitive is `useField`, which registers a single field against a form
context and owns that field's value, meta (`touched`, `dirty`, `valid`,
`pending`), and error message. `useForm` creates the context (provided via Vue's
`provide`/`inject`) and aggregates fields into `values`, `errors`, and
form-level meta. The `Field`/`Form` components are wrappers that call these
composables internally — which is why the two styles interoperate but also why
behavior described for one applies to the other.

Validation is reactive and event-driven, not a single blocking pass. Each field
validates on its own triggers (input, blur, form submit) and validation can be
synchronous or return a Promise; `pending` tracks in-flight async validation.
Because everything is Vue reactivity, `errors` and `meta` update as a side
effect of value changes rather than through an explicit validate call, though
`validate()` and `validateField()` exist for imperative use.

Schema handling is adapter-based. `@vee-validate/yup` and `@vee-validate/zod`
wrap a schema with `toTypedSchema`, which produces both the runtime validator
and TypeScript types for `values`, so `useForm<Schema>()` gets inferred field
names and value types. Built-in rules live in a separate `@vee-validate/rules`
package (25+ common rules), and i18n messages in `@vee-validate/i18n` (40+
locales). The monorepo splits these deliberately so an app that uses Zod pulls
in none of the built-in-rules machinery.

The historical arc explains odd corners. v2 was a Vue 2 directive (`v-validate`)
that attached to inputs and read rules from string expressions. v3 replaced that
with renderless scoped-slot components (`ValidationProvider`,
`ValidationObserver`) — more explicit, tree-shakeable, still Vue 2. v4 threw both
out for Vue 3's Composition API, and its `Form`/`Field` component names and shape
are consciously modeled on React's Formik[^4]. Nested-path typing borrows ideas
from react-hook-form[^4].

## Production Notes

**Validation timing is a frequent surprise.** When a field validates
(`validateOnBlur`, `validateOnChange`, `validateOnInput`, `validateOnModelUpdate`)
is configurable, and the defaults differ between the component API and the
composition API's `defineField`. Teams that report "errors show too early" or
"errors don't clear" are almost always fighting these flags rather than a bug.
Pin the timing config explicitly rather than relying on defaults.

**Typed schemas are the intended path, and mixing styles fights the types.**
Full TypeScript inference flows from `toTypedSchema` into `useForm`. Hand-writing
`validationSchema` as a plain object of functions works but gives you weaker
typing, and combining a typed schema with per-field `rules` props can produce
value types that disagree with runtime behavior. Choose one source of truth per
form.

**Field arrays and nested paths have edge cases.** `useFieldArray` and dotted /
bracketed field names (`users[0].email`) are supported but are the most common
source of reactivity bugs — keying array items correctly and not reusing indices
as keys matters. Deeply dynamic forms are where the abstraction leaks most.

**Bundle cost is real but controllable.** The core is small; the weight comes
from what you attach. Pulling in Yup or Zod adds a schema library to your bundle
(Zod in particular is not tiny), and `@vee-validate/rules` + `@vee-validate/i18n`
add up if you import them wholesale instead of the specific rules you use.

**Version pinning across the Vue 2/3 line.** Do not upgrade a Vue 2 app's
vee-validate 3 to 4 expecting a migration path — it is a rewrite, not an upgrade.
The v4→v5 step is within Vue 3 and far smaller, but still read the changelog for
adapter and default changes.

## When to Use / When Not

**Use when:**
- You're on Vue 3 and want form state (errors, arrays, submission, dirty/touched)
  without hand-rolling it.
- You already validate with Yup or Zod and want that schema to drive the form and
  its types.
- You want to keep full control of markup and use any UI component library.

**Avoid when:**
- You want a batteries-included form builder with inputs, layout, and styling —
  that is not this library's job.
- You're still on Vue 2 with no upgrade plan; you're locked to the older, no
  longer actively developed v2/v3 line.
- Your validation is trivial (one or two fields) — native HTML constraint
  validation may be enough and dependency-free.

## Alternatives

- vuelidate/vuelidate — model-based Vue validation with no components; use when
  you want validation decoupled from templates and don't need form-state helpers.
- formkit/formkit — full form framework with inputs, layout, and validation
  bundled; use when you want the whole form UI, not just validation.
- TanStack/form — framework-agnostic (Vue/React/Solid) headless form state; use
  when you want one form API shared across frameworks.
- colinhacks/zod / jquense/yup — schema libraries, not form libraries; use
  alongside vee-validate rather than instead of it.
- Native HTML5 constraint validation — use for simple forms where a dependency
  isn't justified.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | First release for Vue 2 as a validation directive[^1]. |
| 2.x | ~2017 | `v-validate` directive era; Vue 2, string-expression rules. |
| 3.x | ~2019 | Renderless `ValidationProvider`/`ValidationObserver` components; Vue 2, tree-shakeable[^3]. |
| 4.x | ~2021 | Full Vue 3 / Composition API rewrite; `useForm`/`useField`, Formik-style `Form`/`Field`, typed Yup/Zod adapters[^3][^4]. |
| 5.x | current | Vue 3 line successor to v4; incremental over the v4 model[^2]. |

## References

[^1]: logaretm/vee-validate repository, created 2016-07-30. https://github.com/logaretm/vee-validate
[^2]: vee-validate documentation and demos (current). https://vee-validate.logaretm.com/v5/
[^3]: Vue version support table, project README — v2/v3 target Vue 2, v4/v5 target Vue 3. https://github.com/logaretm/vee-validate#vue-version-support
[^4]: Credits section, project README — v4 API inspired by Formik, nested-path types by react-hook-form. https://github.com/logaretm/vee-validate#credits

## Tags

typescript, vue, form-validation, forms, validation, composition-api, zod, yup, frontend, javascript
