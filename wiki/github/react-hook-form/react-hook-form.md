# react-hook-form/react-hook-form

React Hook Form — performant, flexible form state management for React with minimal re-renders.

## What it is

A TypeScript hook-based form library for React (web + React Native). Optimizes for "uncontrolled inputs" semantics — registers DOM inputs via refs rather than tracking state in React, which minimizes re-renders during typing. The default choice for forms in modern React projects.

## Key features

- `useForm` hook + `register`/`handleSubmit` API.
- Resolver integrations: Zod, Yup, Joi, Valibot, ArkType.
- React Native support.
- Minimal re-renders by default.
- Built-in validation with custom validators.
- TypeScript-first.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- npm package: `react-hook-form`.

## When to reach for it

- You're building forms in React and want minimal re-renders.
- You're integrating with Zod / Yup schema validation.

## When *not* to reach for it

- You want controlled-input semantics — Formik or Final Form fit that better.
- Simple form, no validation — plain useState is fine.

## Maturity signal

45k stars, 2.4k forks, MIT, actively maintained. Open issues = 77, signaling tight maintenance.

## Alternatives

- Formik — controlled-input alternative.
- TanStack Form — TanStack ecosystem fit.
- Conform — newer Remix-flavored form library.

## Tags

react, typescript, forms, library, mit-license, hooks, validation, react-native
