# Mock Test Platform Architecture

## Goals

- Build a scalable mock test platform on Next.js App Router with TypeScript and Tailwind CSS.
- Support credentials-based authentication, timed tests, result history, premium test payments, and admin test management.
- Keep the system Vercel-friendly by using server-first rendering, explicit runtime boundaries, and a PostgreSQL schema that works well with Prisma.

## Architecture Principles

- Server-first by default: use React Server Components for data-heavy pages and only opt into client components for timers, form UX, and payment checkout.
- Secure scoring: never send `correctAnswer` to the browser during an active attempt; submission and scoring happen on the server.
- Stable result history: store per-question attempt rows and a question snapshot on each attempt so historical scores remain valid if a test changes later.
- Clear boundaries: keep route handlers for external integrations and real-time style interactions, and keep reusable business logic in `lib`.
- Vercel-ready: Prisma runs on the Node.js runtime, webhook routes stay server-only, and public content can use ISR where appropriate.

## Application Layers

### 1. Presentation Layer

- `app/` contains pages, layouts, route handlers, metadata, and route protection boundaries.
- `components/` contains reusable UI grouped by domain.
- Tailwind handles layout with flexbox and grid only.

### 2. Domain / Service Layer

- `lib/auth/` manages NextAuth config, password hashing, and authorization helpers.
- `lib/actions/` holds server actions for admin CRUD and authenticated mutations.
- `lib/data/` contains read-model queries for dashboard, test catalog, results, and admin views.
- `lib/payments/` handles Razorpay order creation, signature verification, and premium access checks.
- `lib/validators/` centralizes Zod schemas for request validation.

### 3. Data Layer

- PostgreSQL is the system of record.
- Prisma is the ORM and schema authority.
- Attempt answers are normalized into a dedicated table for auditing and analytics.
- Successful payment records act as the unlock source for premium tests.

## Route Map

| Route | Access | Purpose |
| --- | --- | --- |
| `/` | Public | Landing page, featured tests, CTA |
| `/login` | Public | Credentials login |
| `/signup` | Public | User registration |
| `/dashboard` | Authenticated | User summary, recent attempts, unlocked tests |
| `/dashboard/attempts` | Authenticated | Previous attempts and scores |
| `/dashboard/payments` | Authenticated | Payment history |
| `/test/[id]` | Public / gated by test type | Test detail page, metadata, premium lock state |
| `/test/[id]/instructions` | Authenticated for attempt start | Rules, duration, question count, start action |
| `/test/[id]/start` | Authenticated + access check | Timed test UI |
| `/result/[attemptId]` | Authenticated owner/admin | Final score and detailed result |
| `/admin` | Admin only | Admin dashboard |
| `/admin/tests` | Admin only | Test listing and management |
| `/admin/tests/new` | Admin only | Create a new test |
| `/admin/tests/[id]/edit` | Admin only | Edit test and questions |
| `/api/auth/[...nextauth]` | Public | NextAuth route handler |
| `/api/tests/[id]/start` | Authenticated | Create an attempt and snapshot questions |
| `/api/tests/[id]/submit` | Authenticated | Validate answers, score server-side, finish attempt |
| `/api/payments/order` | Authenticated | Create Razorpay order for a premium test |
| `/api/payments/verify` | Authenticated | Verify client-returned Razorpay signature |
| `/api/webhooks/razorpay` | Razorpay | Server-to-server payment confirmation |

## Folder Structure

```text
app/
  api/
    auth/
      [...nextauth]/
        route.ts
    payments/
      order/
        route.ts
      verify/
        route.ts
    tests/
      [id]/
        start/
          route.ts
        submit/
          route.ts
    webhooks/
      razorpay/
        route.ts
  admin/
    page.tsx
    tests/
      page.tsx
      new/
        page.tsx
      [id]/
        edit/
          page.tsx
  dashboard/
    layout.tsx
    page.tsx
    attempts/
      page.tsx
    payments/
      page.tsx
  login/
    page.tsx
  result/
    [attemptId]/
      page.tsx
  signup/
    page.tsx
  test/
    [id]/
      page.tsx
      instructions/
        page.tsx
      start/
        page.tsx
  layout.tsx
  page.tsx
  robots.ts
  sitemap.ts

components/
  admin/
  auth/
  dashboard/
  layout/
  test/
  ui/

lib/
  actions/
    auth-actions.ts
    payment-actions.ts
    test-actions.ts
  auth/
    auth.ts
    config.ts
    guards.ts
    password.ts
  data/
    attempts.ts
    dashboard.ts
    tests.ts
  payments/
    access.ts
    razorpay.ts
  validators/
    attempt.ts
    auth.ts
    payment.ts
    test.ts
  prisma.ts

prisma/
  schema.prisma
  seed.ts

utils/
  constants.ts
  format.ts
  seo.ts
  time.ts

middleware.ts
```

## Core Domain Flows

### Authentication

1. User signs up with email and password.
2. Password is hashed before persistence.
3. NextAuth uses a credentials provider and JWT session strategy.
4. Middleware and server-side guards protect `/dashboard`, `/result`, and `/admin`.
5. Admin access is enforced through the `role` field on `User`.

### Test Attempt Lifecycle

1. User opens `/test/[id]` and sees public metadata.
2. If the test is premium, access is checked from successful payment records.
3. Starting the test calls `/api/tests/[id]/start`, which creates an `Attempt`, snapshots the question set, and sets `expiresAt`.
4. The client timer is only a UX layer; the server also enforces expiry using `expiresAt`.
5. On submit, the server compares submitted options against canonical answers, stores `AttemptAnswer` rows, calculates score, and marks the attempt as submitted.
6. Result pages read only from stored attempt data, not from mutable live test data.

### Payment Lifecycle

1. User clicks unlock on a premium test.
2. `/api/payments/order` creates a Razorpay order and persists a `Payment` row with status `CREATED`.
3. Client opens Razorpay Checkout using the public key.
4. `/api/payments/verify` or the webhook verifies the signature and marks the payment as `PAID`.
5. Premium access checks rely on a successful payment for the `(userId, testId)` pair.

## Database Design Summary

### Required Core Tables

- `User`: stores identity, hashed password, and role.
- `Test`: stores metadata, publication state, premium flags, pricing, and author.
- `Question`: stores question body, four options, and the correct answer.
- `Attempt`: stores a timed user test session, score summary, and question snapshot.
- `Payment`: stores Razorpay order and payment lifecycle data.

### Supporting Table

- `AttemptAnswer`: stores one row per answered question so prior attempts, breakdowns, and analytics remain queryable.

## Security Rules

- Protect dashboard and admin routes with middleware plus server-side role checks.
- Never expose `correctAnswer` in test-taking payloads.
- Score only on the server from authoritative question data.
- Verify every Razorpay payment signature server-side before unlock.
- Use question snapshots so test edits do not rewrite historical results.

## SEO Strategy

- Published tests should have a unique `slug` stored in the database.
- `generateMetadata` should use `title`, `description`, and canonical test URL data.
- Public test pages can be server-rendered with revalidation for SEO.
- `sitemap.ts` should include only published tests.
- Dashboard, admin, and result pages stay dynamic and private.

## Vercel Deployment Notes

- Use the Node.js runtime for Prisma and Razorpay routes.
- Keep a Prisma singleton in `lib/prisma.ts` to avoid excess client creation during local reloads.
- Use a pooled PostgreSQL connection string in production.
- Mark personalized routes as dynamic and avoid caching authenticated responses.
- Make Razorpay webhook handling idempotent by matching on stored order and payment identifiers.

## Environment Variables To Expect

- `DATABASE_URL`
- `AUTH_SECRET`
- `NEXTAUTH_URL`
- `RAZORPAY_KEY_ID`
- `RAZORPAY_KEY_SECRET`
- `NEXT_PUBLIC_RAZORPAY_KEY_ID`

## Implementation Order

1. Initialize Next.js, Tailwind, Prisma, and NextAuth base setup.
2. Implement the Prisma schema and initial migration.
3. Build auth and route protection.
4. Build admin test creation and question management.
5. Build test-taking flow and server-side scoring.
6. Build result history views.
7. Add Razorpay purchase flow and premium gating.
8. Finish SEO, polish UI, and deployment configuration.
