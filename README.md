# Pro Learner

Pro Learner is a course storefront built with Next.js. Visitors can browse courses, sign in, purchase individual courses, or subscribe to a monthly or yearly Pro plan. Course, purchase, and subscription records are stored in Convex; Stripe processes payments.

## Features

- Browse a course catalog and view course details.
- Sign in and sign up using Clerk.
- Purchase individual courses or subscribe to a monthly or yearly Pro plan through Stripe Checkout.
- Check course access based on an individual purchase or an active subscription.
- View subscription details and open the Stripe Billing Portal.
- Send welcome, purchase confirmation, and subscription activation emails through Resend.
- Limit checkout attempts with Upstash Redis rate limiting (three attempts per user per minute).

The catalog is loaded from the Convex `courses` table. Although `courseData.json` contains example course data, the application does not import or seed it automatically. Populate the Convex table before expecting courses to appear.

## Tech Stack

| Area | Technology |
| --- | --- |
| Web application | Next.js 16 App Router, React 19, TypeScript |
| Styling and UI | Tailwind CSS 4, Base UI, project UI components, Lucide icons, Sonner |
| Authentication | Clerk |
| Database and backend functions | Convex |
| Payments and billing | Stripe |
| Email | Resend and React Email |
| Rate limiting | Upstash Redis and `@upstash/ratelimit` |
| Linting and formatting | Biome |

## Project Structure

| Path | Responsibility |
| --- | --- |
| `app/` | App Router pages, loading states, and API routes. Includes the home page, course catalog/details, Pro plans, billing, and checkout success pages. |
| `app/api/create-billing-portal/` | Authenticated route that creates a Stripe Billing Portal session. |
| `app/api/webhooks/stripe/` | Stripe webhook endpoint for course purchases and subscription events. |
| `components/` | Shared navigation, purchase controls, loading UI, providers, and reusable UI components. |
| `convex/` | Convex schema, queries, mutations, actions, auth configuration, and Clerk HTTP webhook. |
| `emails/` | React Email templates for welcome, course purchase, and Pro activation messages. |
| `lib/` | Stripe and Resend clients, checkout rate limiting, and shared utilities. |
| `constants/` | Pro plan display data. |
| `courseData.json` | Example course records; not automatically loaded into Convex. |
| `package.json` | npm scripts and dependencies. |
| `next.config.ts` | Next.js configuration, including the allowed YouTube thumbnail image host. |
| `biome.json` | Biome formatter and linter configuration. |

## Getting Started

Clone the repository from its GitHub page, then run these commands from the project directory. The checked-in `package-lock.json` indicates npm is the package manager.

```bash
npm ci
```

Configure the integrations described in [Environment Variables](#environment-variables), then initialize or select a Convex deployment and start its development process:

```bash
npx convex dev
```

In a second terminal, start the Next.js development server:

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Convex functions and the schema are deployed to the selected development deployment by the Convex CLI. Course records still need to be added separately; this repository does not include a seed command.

## Environment Variables

The application integrates with external Clerk, Convex, Stripe, Resend, and Upstash services. The following names are read directly by the code. Keep secret values out of source control; `.gitignore` excludes `.env*` files.

| Variable | Used by | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_APP_URL` | Next.js routes and Convex functions | Base URL used for checkout return URLs and email links. For local development, use the local app origin. |
| `NEXT_PUBLIC_CONVEX_URL` | Next.js application | URL of the Convex deployment used by the client and server-side queries. |
| `CLERK_JWT_ISSUER_DOMAIN` | Convex auth configuration | Clerk issuer domain used to validate Convex authentication. Configure it for the selected Convex deployment. |
| `CLERK_WEBHOOK_SECRET` | Convex HTTP action | Verifies Clerk webhook requests. |
| `STRIPE_SECRET_KEY` | Next.js and Convex | Authenticates Stripe API requests. |
| `STRIPE_WEBHOOK_SECRET` | Next.js Stripe webhook route | Verifies Stripe webhook signatures. |
| `STRIPE_MONTHLY_PRICE_ID` | Convex Stripe action | Stripe Price ID used for the monthly Pro subscription. |
| `STRIPE_YEARLY_PRICE_ID` | Convex Stripe action | Stripe Price ID used for the yearly Pro subscription. |
| `RESEND_API_KEY` | Convex webhook actions | Sends welcome, purchase confirmation, and Pro activation emails. |
| `UPSTASH_REDIS_REST_URL` | Convex checkout action | Upstash Redis REST endpoint used by checkout rate limiting. |
| `UPSTASH_REDIS_REST_TOKEN` | Convex checkout action | Credential for the Upstash Redis REST endpoint. |

Set `NEXT_PUBLIC_CONVEX_URL`, `NEXT_PUBLIC_APP_URL`, and the Stripe webhook secret in the Next.js environment. Set the variables consumed by Convex functions in the environment for the selected Convex deployment as well; `STRIPE_SECRET_KEY` and `NEXT_PUBLIC_APP_URL` are used in both runtimes. Clerk's Next.js integration also requires the credentials for your Clerk application; follow Clerk's setup instructions for the installed SDK.

## Available Scripts

Run scripts with `npm run <script>`:

| Script | Command | Description |
| --- | --- | --- |
| `dev` | `next dev` | Start the Next.js development server. |
| `build` | `next build` | Build the production application. |
| `start` | `next start` | Serve a production build. |
| `lint` | `biome check` | Check supported project files with Biome. |
| `format` | `biome format --write` | Format supported project files in place. |

## Architecture / How It Works

The App Router renders the course, Pro, and billing screens. Client components use Clerk for identity and Convex React hooks for live queries and actions; server-rendered pages can query Convex through its HTTP client. Convex stores users, courses, purchases, and subscription state.

Clerk middleware handles application requests, and Convex is configured to validate Clerk identities. Clerk's `/clerk-webhook` HTTP action creates a Stripe customer and Convex user when a user is created, and processes user updates and deletions. Stripe Checkout sessions are created by Convex actions. The Stripe webhook records course purchases and handles subscription lifecycle events; the billing portal route creates a portal session for the signed-in user's Stripe customer. Welcome and transaction emails are sent through Resend.

Access to a course is granted when the signed-in user has purchased that course or has an active subscription. Checkout actions apply a per-user Upstash rate limit of three requests per 60 seconds.

## API

| Method and path | Purpose | Request and response |
| --- | --- | --- |
| `POST /api/create-billing-portal` | Create a Stripe Billing Portal session for the authenticated user. | No request body. Returns `{ "url": "..." }` on success; returns an `error` message with HTTP 401 when unauthenticated, 404 when the user or Stripe customer is missing, or 500 on an internal error. |
| `POST /api/webhooks/stripe` | Receive Stripe webhook events. | Send the raw Stripe event with a valid `stripe-signature` header. Supported events: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, and `customer.subscription.deleted`. Returns `{ "received": true }` when handled, HTTP 400 for an invalid signature, or HTTP 500 when handling fails. |
| `POST /clerk-webhook` (Convex HTTP endpoint) | Process Clerk user webhooks. | Requires a valid Svix-signed Clerk webhook request. Handles `user.created`, `user.updated`, and `user.deleted`; returns a plain-text status response. |

## Database

Convex is the database and backend platform. Its schema defines:

| Table | Stored data |
| --- | --- |
| `users` | Clerk ID, email, optional name, Stripe customer ID, and optional current subscription reference. |
| `courses` | Title, description, image URL, and price. |
| `purchases` | User and course references, Stripe checkout session ID, amount, and purchase timestamp. |
| `subscriptions` | User and Stripe subscription IDs, monthly/yearly plan, billing period, status, and cancel-at-period-end flag. |

Indexes support lookups by Clerk ID, Stripe customer ID, current subscription, Stripe subscription ID, and user/course pair. Run `npx convex dev` to deploy the schema and functions to a development deployment. The repository contains no migration or course-seeding script.

## Authentication

Clerk provides sign-in, sign-up, and user session UI. Clerk middleware is configured in `proxy.ts`; the Convex client provider passes Clerk authentication to Convex, whose auth configuration uses the Clerk issuer domain. A Clerk webhook keeps the Convex user record and associated Stripe customer in sync with user creation, updates, and deletion.

## Deployment

No Docker files or platform-specific deployment configuration are included. A production setup needs a Next.js deployment and a Convex deployment, with the relevant environment variables configured in each runtime. Configure Clerk and Stripe webhook destinations to reach the routes documented above, and populate the Convex `courses` table before launch.

## Contributing

1. Create a branch for your change.
2. Install dependencies with `npm ci` and configure the required services for the behavior being changed.
3. Run `npm run lint` and `npm run build` before opening a pull request.
4. Include a concise description of the change and any setup or verification steps in the pull request.
