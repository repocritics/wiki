# calcom/cal.diy

Cal.com / "cal.diy" — the open-source Calendly alternative. Scheduling infrastructure that you can self-host or use as the hosted SaaS.

## What it is

A TypeScript-based scheduling app: users connect a calendar (Google Calendar, Office 365, Apple Calendar), define availability rules + event types, and share booking links. Recipients pick a slot; the app creates the calendar event, sends invites, and handles reminders. T3-stack architecture (Next.js + tRPC + Prisma + Tailwind). Hosted at cal.com (recently rebranded as cal.diy in some surfaces) or self-hostable.

## Key features

- Calendar connectors: Google, Office 365, Apple/iCloud, CalDAV, Zoom, Teams, Daily, Webex.
- Event types: 1:1, group, round-robin, collective, managed.
- Routing forms — direct bookings to the right team member based on form answers.
- Payments via Stripe / Razorpay for paid bookings.
- Embed widgets for any website.
- White-label / enterprise team features in the paid tier.
- MIT-licensed.

## Tech stack

- TypeScript primary; T3 stack (Next.js + tRPC + Prisma + Zod + NextAuth + Tailwind).
- Turborepo monorepo.
- Postgres for data storage.

## When to reach for it

- You want a Calendly alternative without the Calendly bill.
- You need branded scheduling infrastructure as part of your own app.
- You want self-hosted scheduling for compliance / data-residency reasons.

## When *not* to reach for it

- You want zero-setup hosted scheduling — Calendly itself is faster to start.
- You want to avoid operating a Postgres + Next.js + Redis stack.

## Maturity signal

45k stars, 14k forks, MIT, actively maintained. 4+ years under Cal.com Inc.

## Alternatives

- Calendly — commercial, fully managed.
- SavvyCal — Calendly competitor with stronger team scheduling.
- Rallly — use for poll-style group scheduling.

## Tags

typescript, nextjs, scheduling, calendar, self-hosted, t3-stack, prisma, mit-license, trpc
