# Basic Diet

> A connected diet-subscription, meal-ordering, and operations platform built around a customer mobile application, an internal operations dashboard, and a shared backend that owns the platform’s business rules.

Basic Diet is not a single application. It is a multi-repository product ecosystem designed around two very different user groups:

- **Customers**, who interact with the service through the Flutter mobile application.
- **Internal operations staff**, who manage subscriptions, customers, meals, orders, fulfilment, payments, accounting, and day-to-day administration through the web dashboard.

Both applications depend on the same backend API. That backend is deliberately treated as the authoritative layer for persistence, authentication, permissions, subscription state, checkout rules, pricing, entitlements, ordering, payment-related behavior, and other business-critical transitions.

This organization keeps those responsibilities separated instead of trying to place customer UX, staff operations, and sensitive business logic in one codebase.

---

## Table of Contents

- [What Basic Diet Is](#what-basic-diet-is)
- [Product Goals](#product-goals)
- [Who the Platform Serves](#who-the-platform-serves)
- [Platform Architecture](#platform-architecture)
- [Repository Map](#repository-map)
- [Operations Dashboard](#operations-dashboard)
  - [Dashboard Responsibilities](#dashboard-responsibilities)
  - [Dashboard Business Areas](#dashboard-business-areas)
  - [Dashboard Architecture](#dashboard-architecture)
  - [Dashboard Technology](#dashboard-technology)
  - [Dashboard Quality Workflow](#dashboard-quality-workflow)
- [Backend API](#backend-api)
  - [Backend Responsibilities](#backend-responsibilities)
  - [Backend Domain Model](#backend-domain-model)
  - [Backend Architecture](#backend-architecture)
  - [Authentication and Security](#authentication-and-security)
  - [Subscriptions and Entitlements](#subscriptions-and-entitlements)
  - [Menu, Catalog, and Meal Builder](#menu-catalog-and-meal-builder)
  - [Orders, Checkout, and Fulfilment](#orders-checkout-and-fulfilment)
  - [Payments and Accounting](#payments-and-accounting)
  - [Operational Scripts](#operational-scripts)
  - [Backend Testing and Release Gates](#backend-testing-and-release-gates)
- [Mobile Application](#mobile-application)
  - [Customer Journeys](#customer-journeys)
  - [Mobile Architecture](#mobile-architecture)
  - [Mobile Technology](#mobile-technology)
- [How the Applications Work Together](#how-the-applications-work-together)
- [Important Business Flows](#important-business-flows)
  - [Subscription Flow](#subscription-flow)
  - [Meal and Catalog Flow](#meal-and-catalog-flow)
  - [One-Time Order Flow](#one-time-order-flow)
  - [Delivery and Pickup Flow](#delivery-and-pickup-flow)
  - [Accounting and Payment Flow](#accounting-and-payment-flow)
- [Data and State Ownership](#data-and-state-ownership)
- [Permissions and Security Boundaries](#permissions-and-security-boundaries)
- [Media and External Services](#media-and-external-services)
- [Localization and Arabic-First UX](#localization-and-arabic-first-ux)
- [Testing Strategy](#testing-strategy)
- [Operational Safety](#operational-safety)
- [Engineering Conventions](#engineering-conventions)
- [Documentation Strategy](#documentation-strategy)
- [Development and Release Model](#development-and-release-model)
- [Project Status](#project-status)
- [Repository Guide for Reviewers](#repository-guide-for-reviewers)

---

## What Basic Diet Is

Basic Diet is a software platform for running a diet-subscription and meal-ordering business.

The product has to solve more than a normal “menu + checkout” problem. The system works with customer accounts, subscription plans, recurring meal access, add-ons, premium meals, one-time orders, fulfilment, pickup, delivery, payments, accounting-oriented records, operational adjustments, internal users, and business rules that can affect more than one client at the same time.

That is why the platform is split into dedicated applications rather than being implemented as a single monolithic frontend.

At a high level:

```text
Customer experience                    Internal operations
        │                                      │
        ▼                                      ▼
Flutter Mobile App                  React Operations Dashboard
        │                                      │
        └──────────────────┬───────────────────┘
                           │
                           ▼
                  Shared Backend API
              Express + MongoDB/Mongoose
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         Persistence   Media/Files   Notifications/
                                   external services
```

The backend is the shared authority. The dashboard and mobile app do not independently decide business-critical rules when those rules affect shared state, money, subscription eligibility, entitlements, fulfilment, or security.

---

## Product Goals

The platform is structured to support several business goals at the same time.

### Customer goals

Customers need to be able to:

- Create and access accounts.
- Move through onboarding and verification.
- Discover available plans.
- Browse meals and products.
- Work with cart/order flows.
- Use customer-facing subscription experiences.
- Review orders and account information.
- Access a localized mobile experience.
- Maintain relevant preferences across sessions.

### Operations goals

Internal users need to be able to:

- Find and manage customers.
- Create and maintain subscriptions.
- Work with packages and plan configuration.
- Operate the meal/catalog system.
- Manage add-ons and premium meals.
- Process one-time orders.
- Work with delivery and pickup.
- Review payment and accounting-related data.
- Manage dashboard users.
- Apply controlled manual operational actions.
- Work with promo codes and notifications.
- Access only the areas permitted for their role.

### Engineering goals

The system also needs to:

- Keep one authoritative representation of business rules.
- Prevent frontend clients from becoming independent sources of truth.
- Keep dashboard and mobile contracts compatible.
- Protect security-sensitive endpoints.
- Test concurrency-sensitive operations.
- Make operational scripts explicit and reviewable.
- Support safe data migrations, bootstrap processes, and maintenance.
- Keep application-specific documentation close to each codebase.

---

## Who the Platform Serves

Basic Diet has three broad classes of system actors.

### Customers

Customers primarily use the mobile application.

Their experience includes areas such as:

- Onboarding
- Authentication
- Verification and password flows
- Plans
- Home
- Menu
- Cart
- Orders
- Profile
- Localized content and navigation

### Dashboard Staff

Internal staff use the operations dashboard.

The exact actions available to a staff account depend on the backend’s access policies and the role assigned to that user. Dashboard areas include customer, subscription, menu, operations, payment, accounting, fulfilment, and staff-management workflows.

### System / Operations Maintainers

Developers and technical operators interact with the backend’s maintenance, validation, seed, migration, audit, repair, and release-check scripts.

This is an important distinction: some backend capabilities are intentionally not normal user-facing API actions. They exist to support safe maintenance and handover.

---

## Platform Architecture

The core architecture is client/server with two clients sharing one backend.

```text
┌──────────────────────────────────────────────────────────────┐
│                     BASIC DIET PLATFORM                      │
└──────────────────────────────────────────────────────────────┘

        CUSTOMER SIDE                     OPERATIONS SIDE

┌───────────────────────────┐      ┌──────────────────────────────┐
│ Flutter Mobile App        │      │ React Operations Dashboard   │
│                           │      │                              │
│ • onboarding              │      │ • customers                  │
│ • authentication          │      │ • subscriptions              │
│ • plans                   │      │ • menu/catalog               │
│ • menu                    │      │ • orders                     │
│ • cart                    │      │ • delivery / pickup          │
│ • orders                  │      │ • payments / accounting      │
│ • profile                 │      │ • staff / roles              │
└──────────────┬────────────┘      └───────────────┬──────────────┘
               │                                   │
               │            REST / API             │
               └─────────────────┬─────────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Backend API             │
                    │ Node.js + Express       │
                    │ MongoDB + Mongoose      │
                    │                         │
                    │ • auth                  │
                    │ • business rules        │
                    │ • subscriptions         │
                    │ • checkout              │
                    │ • catalog               │
                    │ • entitlements          │
                    │ • fulfilment            │
                    │ • payments              │
                    │ • operations            │
                    └────────────┬────────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               │                 │                 │
               ▼                 ▼                 ▼
          MongoDB           Cloudinary        Firebase Admin
        persistence           media            integration
```

The architecture intentionally keeps clients focused on presentation, input handling, interaction, and synchronization with server state.

---

## Repository Map

| Repository | Role | Main Technology | Primary Audience |
| --- | --- | --- | --- |
| [`Basic-Diet/client_dashbourd`](https://github.com/Basic-Diet/client_dashbourd) | Internal operations dashboard | React, TypeScript, Vite, TanStack | Operations staff |
| [`Basic-Diet/backend`](https://github.com/Basic-Diet/backend) | Shared API and authoritative business layer | Node.js, Express, MongoDB/Mongoose | Both clients / system |
| [`Basic-Diet/mobile_app`](https://github.com/Basic-Diet/mobile_app) | Customer-facing mobile app | Flutter, Dart, BLoC | Customers |
| [`Basic-Diet/documentations`](https://github.com/Basic-Diet/documentations) | Shared engineering and workflow documentation | Markdown / workflow docs | Developers / maintainers |

Each product repository contains its own README with implementation-specific details. This organization profile explains how the pieces fit together.

---

# Operations Dashboard

The operations dashboard is the internal web application used to run the business.

It is not a generic admin template. Its route structure reflects actual operational domains, and many screens are tied directly to backend contracts that control subscriptions, fulfilment, catalog behavior, payments, permissions, and business-state transitions.

## Dashboard Responsibilities

The dashboard is responsible for:

- Presenting internal business data.
- Giving staff safe interfaces for operational actions.
- Validating user input before it is sent to the backend.
- Keeping remote/server state synchronized.
- Rendering loading, empty, error, and success states.
- Restricting routes and UI based on the current user context.
- Supporting Arabic-oriented business administration.
- Making complex workflows understandable without copying authoritative backend rules into UI code.

The dashboard does **not** become the final authority for permission checks, payment rules, subscription rules, or entitlement balances. Those remain backend responsibilities.

---

## Dashboard Business Areas

The current protected route organization includes dedicated areas for multiple operational domains.

### Dashboard and overview

The main dashboard provides internal visibility into the current system and operational state.

### Accounting

Accounting-oriented interfaces expose data needed for financial and day-to-day operational review.

The frontend can display and organize this data, while authoritative monetary calculations and stored records belong to backend contracts.

### Add-ons

Add-ons are part of the wider subscription and meal ecosystem.

The platform contains backend tests for:

- Add-on entitlement authority
- Credit allocation
- Add-on credit lifecycle
- Dynamic add-on groups
- Dashboard/mobile parity
- Multi-user cases
- Display-key backfills
- Subscription add-on selection contracts

This is a good example of why the dashboard cannot treat add-ons as simple local form state.

### Dashboard users

Internal users can be administered through dedicated dashboard-user flows.

Backend policies cover:

- Staff-user behavior
- User search
- Menu/route roles
- Kitchen/read-only permissions
- Superadmin/bootstrap processes

### Delivery

Delivery has a dedicated protected dashboard area and corresponding backend fulfilment contracts.

### Manual deductions

Manual operational adjustments are isolated into their own workflow rather than being hidden inside unrelated screens.

The backend has dedicated validation/tests for manual deductions and pickup-related behavior.

### Menu

The menu/catalog area is one of the more complex parts of the platform.

It intersects with:

- Meal planner behavior
- Meal Builder
- Production products
- Menu identity
- Menu identity suggestions
- Premium meals
- Weight-based pricing
- One-time menu/catalog behavior
- Dashboard/mobile parity

### Notifications

Notifications have their own operational route area and are treated as a platform concern rather than being embedded randomly into individual pages.

### One-time orders

The system supports one-time ordering in addition to subscription-oriented workflows.

Backend coverage includes both operations-level tests and full-flow tests.

### Operations

Operations routes provide internal views tied to the day-to-day running of the service.

### Packages

Packages and plans are managed separately from individual subscription instances.

### Payments

Payment administration is separated from subscription and accounting screens so payment-related data can be reviewed through focused workflows.

### Pickup branches

Pickup locations/branches are explicitly represented as a domain.

The backend includes scripts and tests for:

- Ensuring a default pickup location
- Auditing pickup address serialization
- Pickup behavior in order/fulfilment flows

### Premium meals

Premium meals are treated as a complete lifecycle rather than a single product flag.

The backend release gates include premium-meal lifecycle coverage and premium salad eligibility policies.

### Profile

Dashboard users have a dedicated profile area.

### Promo codes

Promotional configuration is exposed through a dedicated protected area.

### Subscriptions

Subscriptions are a central product domain and connect to:

- Plan selection
- Pricing
- Creation
- Current-subscription resolution
- Day modifications
- Balance policy
- Freeze/extension-related behavior
- Add-ons
- Fulfilment
- Checkout
- Accounting
- Mobile contracts

---

## Dashboard Architecture

The dashboard is a React/Vite application with a clear separation between routing, reusable UI, shared utilities, types, hooks, and application-level providers.

High-level source organization:

```text
src/
├── App.tsx
├── components/
├── constants/
├── features/
├── hooks/
├── lib/
├── providers/
├── routes/
│   ├── __root.tsx
│   ├── index.tsx
│   └── _protected/
├── types/
├── utils/
├── router.tsx
├── routeTree.gen.ts
├── main.tsx
└── index.css
```

The generated route tree comes from TanStack Router’s routing model.

Protected application areas live under the `_protected` route group so access-oriented routing is not scattered across unrelated page components.

---

## Dashboard Technology

### Core

- React 19.2
- React DOM 19.2
- TypeScript 5.9
- Vite 7

### Routing and remote state

- TanStack Router
- TanStack Query
- TanStack Table
- TanStack Query Devtools
- TanStack Router Devtools

### Forms and validation

- React Hook Form
- Zod
- `@hookform/resolvers`

### UI and interaction

- Tailwind CSS 4
- Radix UI
- shadcn tooling
- Lucide React
- Motion
- dnd-kit
- Recharts
- Sonner
- Vaul
- React Day Picker

### Networking and utilities

- Axios
- date-fns
- js-cookie
- class-variance-authority
- clsx
- tailwind-merge

### Testing and tooling

- Vitest
- Testing Library
- ESLint
- Prettier
- TypeScript type checking

---

## Dashboard Quality Workflow

The dashboard exposes explicit project scripts for:

```bash
npm run dev
npm run build
npm run lint
npm run format
npm run test
npm run test:watch
npm run typecheck
npm run preview
```

The production build runs TypeScript project compilation before Vite bundling:

```text
tsc -b
   ↓
vite build
```

That makes type correctness part of the normal production-build path instead of an optional separate check.

---

# Backend API

The backend is the business authority of the platform.

It is responsible for the rules that cannot safely be decided independently by the dashboard or mobile application.

## Backend Responsibilities

The backend covers concerns including:

- Customer authentication
- Dashboard authentication
- User/account behavior
- Dashboard staff users
- Permissions
- Subscription plans
- Subscription lifecycle
- Subscription balances
- Subscription day modifications
- Add-ons and entitlements
- Meal planner
- Meal Builder
- Menu/catalog
- Premium meals
- One-time orders
- Checkout
- Pickup
- Fulfilment
- Payments
- Accounting-oriented endpoints
- Notifications
- Manual deductions
- Operational scripts
- Database indexes
- Validation
- API documentation
- Logging
- Security hardening

---

## Backend Domain Model

The backend is organized into familiar server-side concerns.

```text
src/
├── index.js
├── app.js
├── db.js
├── config/
├── constants/
├── content/
├── controllers/
├── docs/
├── jobs/
├── locales/
├── mappers/
├── middleware/
├── models/
├── routes/
├── scripts/
├── services/
├── types/
└── utils/
```

This separation keeps:

- HTTP routing separate from business services.
- Authentication/security middleware centralized.
- Persistence models distinct from controllers.
- Operational tooling explicit.
- Shared mappings/utilities reusable across domains.

---

## Backend Architecture

A normal protected request follows a pattern similar to:

```text
Client request
      ↓
Express route
      ↓
Security / auth / validation middleware
      ↓
Controller
      ↓
Service / domain logic
      ↓
Mongoose model / integration
      ↓
MongoDB or external service
      ↓
Normalized API response
```

The exact implementation varies by domain, but the principle is consistent: client applications should not bypass backend validation simply because the same rule is also represented in UI state.

---

## Authentication and Security

The backend uses:

- JWT
- bcryptjs
- Helmet
- CORS
- express-rate-limit

Security-related automated checks include areas such as:

- CORS preflight behavior
- Public API surface
- Checkout hardening
- Security hardening
- Webhook security

Authentication and route visibility in the dashboard improve UX, but they are not the final security boundary.

```text
Frontend visibility / route guard
              ≠
       backend authorization
```

The backend must still reject unauthorized operations even if a user manually constructs a request.

---

## Subscriptions and Entitlements

Subscriptions are one of the most complex platform domains.

The backend contains explicit test coverage for concerns including:

- Subscription creation contracts
- Current-subscription resolution
- Subscription balance policy
- Subscription day modification policy
- Fulfilment concurrency
- Checkout invoice concurrency
- Bilingual subscription responses
- Add-on selection policies
- Add-on credit allocation
- Owned add-on entitlement snapshots
- Dynamic add-on groups
- Entitlement authority
- Owned meal entitlement
- Credit release/idempotency
- Dashboard/mobile parity

This suggests a deliberate separation between:

- what the customer or staff user selects,
- what the client displays,
- and what the backend is allowed to commit.

### Conceptual subscription lifecycle

```text
Available plan
     ↓
Customer/staff selection
     ↓
Authoritative backend validation
     ↓
Pricing / entitlement checks
     ↓
Subscription creation
     ↓
Current subscription state
     ↓
Lifecycle operations
  ├─ fulfilment
  ├─ day modifications
  ├─ balance changes
  ├─ add-on usage
  └─ controlled operational adjustments
```

---

## Menu, Catalog, and Meal Builder

The catalog domain is broader than a static list of meals.

The backend includes tooling and tests around:

- Meal planner data
- Dynamic meal planner contracts
- Meal planner card lifecycle
- Dashboard compatibility
- Production products
- Meal Builder complete catalog
- Builder Catalog v2 contracts
- Menu identity mappings
- Menu identity suggestions
- Menu identity suggestion approval
- One-time menu catalog
- Weight-step pricing
- Weight-pricing authority
- Premium meal lifecycle
- Premium salad eligibility
- Catalog consistency
- Dashboard/mobile parity

The repository also includes migration and repair scripts for catalog-related data.

Examples include:

- Syncing option prices
- Migrating builder data to menu data
- Backfilling meal categories
- Cleaning premium catalog data
- Checking catalog health
- Validating menu identity links
- Suggesting menu identity mappings

These scripts make data-shape evolution explicit rather than relying on one-off manual database changes.

---

## Orders, Checkout, and Fulfilment

The backend includes separate coverage for:

### Subscription checkout

- Checkout integration behavior
- Checkout concurrency
- Invoice-concurrency scenarios
- Hardening around sensitive checkout flows

### One-time orders

- Operational behavior
- Full-flow order behavior
- One-time menu catalog compatibility

### Fulfilment

- Fulfilment endpoint contracts
- Fulfilment status
- Subscription fulfilment concurrency
- Delivery/pickup-related operational behavior

Concurrency tests are particularly important here because the same customer/order/subscription may be affected by more than one request or actor.

---

## Payments and Accounting

Payment and accounting behavior is intentionally separated from presentation concerns.

Backend coverage includes:

- Payment initialization logging
- Payment indexes
- Dashboard accounting daily reports
- Payment-related operational investigation
- Release validation that includes payment-sensitive behavior

The dashboard can present totals, reports, statuses, and actions, but the backend remains responsible for the authoritative stored state.

---

## Operational Scripts

The backend contains a large number of explicit scripts for maintenance and production safety.

Examples include:

### Bootstrap

- Bootstrap superadmin
- Bootstrap initial data
- Bootstrap accounts
- Bootstrap new menu

### Seeds

- Subscription plans
- Subscription add-ons
- One-time menu
- Dashboard users
- Legal content
- Demo data

### Audits

- Duplicate active subscriptions
- Pickup address serialization
- Add-on plan/category conflicts
- Premium salad protein eligibility

### Repairs and backfills

- Meal planner primary content
- Weight pricing
- Add-on display keys
- Meal categories
- Option prices

### Validation

- Backend validation
- Staging validation
- Data-integrity validation
- Menu identity validation
- Catalog health

### Database / handover

- Production index creation
- Controlled handover reset checks
- Executable handover reset
- Reset-safety tests

This matters because production maintenance is part of the application lifecycle. It should be scripted, reviewable, repeatable, and tested where possible.

---

## Backend Testing and Release Gates

The backend has a notably broad test surface.

The main `test:release-gates` command combines many focused suites rather than relying on one basic smoke test.

Release-gate coverage includes areas such as:

- Core backend validation
- Meal planner repair
- Database reset safety
- Dashboard staff users
- Bootstrap semantics
- Subscription bilingual contracts
- Security
- Checkout concurrency
- Checkout integration
- One-time orders
- Subscription policies
- Current subscription resolution
- Add-on credit allocation
- Add-on entitlement authority
- Owned meal entitlements
- Add-on credit lifecycle
- Dashboard/mobile parity
- Mobile API contracts
- Payment initialization logging
- Builder Catalog v2
- Dashboard user search
- Kitchen permissions
- Manual deductions
- One-time menu
- Weight pricing
- Premium meal lifecycle

Additional targeted suites cover:

- Account deletion
- Weekly menu dashboard
- Premium salad eligibility
- Catalog consistency
- Meal Builder parity
- Menu parity
- Add-on parity
- Subscription selection policies
- Menu identity behavior
- Staging/data validation

The backend testing toolchain includes:

- Jest
- Mocha
- Chai
- Supertest
- Sinon
- mongodb-memory-server

Different suites can therefore test isolated logic, HTTP behavior, integrations, and database-dependent workflows.

---

# Mobile Application

The mobile application is the customer-facing product.

It is implemented in Flutter and is structurally separate from the internal dashboard.

## Customer Journeys

The mobile app includes customer-facing areas for:

### Onboarding

Users can enter the application through an onboarding experience and localized presentation.

### Authentication

The app includes dedicated flows for:

- Registration
- Login
- Verification
- Forgot password
- Change password
- Password-change completion

### Plans

Customers can discover subscription plans and move into backend-controlled subscription flows.

### Main application

Core customer areas include:

- Home
- Menu
- Cart
- Orders
- Profile

### External content

Where required, the mobile app can use:

- URL launching
- Embedded WebView experiences

---

## Mobile Architecture

The Flutter application is separated into major layers:

```text
lib/
├── app/
├── data/
├── domain/
├── presentation/
├── firebase_options.dart
└── main.dart
```

### `app`

Application-level configuration such as routing, dependency setup, and bootstrap behavior.

### `data`

Remote/local data implementation, serialization, repositories, and persistence concerns.

### `domain`

Domain-facing contracts and abstractions.

### `presentation`

Screens, widgets, state management, and customer-facing features.

This keeps networking and persistence logic from being tightly coupled to Flutter widgets.

---

## Mobile Technology

### Core

- Flutter
- Dart `>=3.7.0 <4.0.0`

### State management

- bloc
- flutter_bloc
- equatable
- RxDart

### Networking and serialization

- Dio
- Retrofit
- json_annotation
- json_serializable
- pretty_dio_logger

### Routing and dependencies

- go_router
- get_it

### Persistence and device-side storage

- flutter_secure_storage
- shared_preferences

### Localization and responsive UI

- easy_localization
- flutter_screenutil
- intl

### Supporting packages

- Firebase Core
- Pinput
- flutter_svg
- fluttertoast
- url_launcher
- webview_flutter
- internet_connection_checker
- uuid
- dartz

### Assets

The app bundles:

- Images
- Icons
- Fonts
- JSON
- Translation files

Configured fonts include Arimo and Tajawal.

---

# How the Applications Work Together

The three primary applications share domain contracts but have different responsibilities.

## Example: customer subscription

```text
Mobile app
  ↓
Customer selects subscription options
  ↓
Mobile validates input shape
  ↓
Backend receives request
  ↓
Backend verifies:
  • allowed plan
  • current subscription state
  • pricing/business policy
  • add-on selections
  • entitlement rules
  ↓
Backend persists authoritative state
  ↓
Mobile displays resulting subscription
  ↓
Dashboard can later operate on the same backend record
```

## Example: dashboard catalog update

```text
Dashboard
  ↓
Staff edits product/menu data
  ↓
React Hook Form + Zod validate expected shape
  ↓
API request
  ↓
Backend validates catalog rules
  ↓
MongoDB state updates
  ↓
Dashboard invalidates/refetches affected queries
  ↓
Mobile reads updated catalog through backend contracts
```

The important point is that both clients converge on the backend rather than synchronizing directly with each other.

---

# Important Business Flows

## Subscription Flow

```text
Plan configuration
      ↓
Customer/staff selection
      ↓
Quote / authoritative validation
      ↓
Subscription creation
      ↓
Current subscription
      ↓
Operational lifecycle
      ├─ fulfilment
      ├─ day modifications
      ├─ add-on usage
      ├─ premium behavior
      └─ manual controlled adjustments
```

Because subscription changes affect shared business state, final decisions are made by the backend.

---

## Meal and Catalog Flow

```text
Catalog / Meal Builder source
      ↓
Backend validates canonical product identity
      ↓
Dashboard manages operational content
      ↓
Mobile consumes customer-ready catalog
      ↓
Orders / subscriptions reference shared identities
```

Identity and parity tests help prevent dashboard/mobile clients from drifting into incompatible representations of the same product.

---

## One-Time Order Flow

```text
Customer order entry
      ↓
Backend validates one-time catalog/order contract
      ↓
Order persisted
      ↓
Operations dashboard receives order
      ↓
Staff works fulfilment state
      ↓
Customer-visible order state updates
```

---

## Delivery and Pickup Flow

```text
Order / subscription fulfilment
      ↓
Delivery or pickup context
      ↓
Backend validates allowed transition
      ↓
Dashboard performs operational action
      ↓
Backend persists result
      ↓
Clients receive current fulfilment state
```

Pickup-location data is explicitly maintained and audited because address/serialization consistency matters to operational workflows.

---

## Accounting and Payment Flow

```text
Checkout / payment event
      ↓
Backend payment-related processing
      ↓
Authoritative record/index state
      ↓
Dashboard payment + accounting views
      ↓
Operational review / reporting
```

The system avoids treating UI totals as the final record of truth.

---

# Data and State Ownership

A key engineering principle in Basic Diet is deciding **where state is authoritative**.

### Backend-owned state

Examples include:

- Authentication authority
- Subscription lifecycle
- Payment-related state
- Entitlement balances
- Checkout outcomes
- Fulfilment state
- Persistent customer/order records
- Role permissions
- Catalog identities

### Dashboard-owned state

Examples are mostly UI concerns:

- Current route
- Open dialogs
- Form values before submission
- Filters
- Table sorting/pagination
- Query loading/error state
- Local interaction state

### Mobile-owned state

Examples include:

- Current navigation
- Local UI state
- Securely stored client tokens/credentials where appropriate
- Preferences
- Cached/localized presentation state

The client applications should synchronize with backend state rather than becoming competing sources of truth.

---

# Permissions and Security Boundaries

The system uses layered access control.

```text
UI visibility
      ↓
Client route protection
      ↓
Authenticated request
      ↓
Backend authentication
      ↓
Backend role/policy checks
      ↓
Allowed domain operation
```

The first two layers improve UX.

The final backend layers provide actual security.

The backend test surface includes policies for:

- Dashboard menu roles
- Staff users
- Kitchen/read-only behavior
- Security hardening
- Public API surface
- Webhooks
- Checkout
- Account deletion

---

# Media and External Services

The backend package includes:

### Cloudinary

Used for media handling alongside Multer.

### Firebase Admin

Available for server-side Firebase-related integration.

### Swagger

`swagger-jsdoc` and `swagger-ui-express` are included for API documentation support.

### Winston

Used for structured logging/observability.

These integrations remain on the server side where secrets and privileged credentials can be protected.

---

# Localization and Arabic-First UX

The platform includes localization concerns across web and mobile.

### Dashboard

The dashboard README describes an Arabic-first administration experience.

### Mobile

The Flutter app includes:

- `easy_localization`
- translation assets
- Tajawal font
- Arimo font
- responsive sizing with `flutter_screenutil`

### Backend

The backend includes a `locales` area and tests for bilingual subscription responses.

Localization therefore exists across product boundaries rather than being a single frontend string file.

---

# Testing Strategy

Basic Diet uses different testing strategies for different risks.

## Dashboard

The dashboard uses:

- Vitest
- Testing Library
- TypeScript checks
- ESLint
- Production build validation

## Backend

The backend contains:

- Focused policy tests
- Integration tests
- Security tests
- Concurrency tests
- Contract tests
- Parity tests
- Operational-safety tests
- Database/in-memory test support
- Release gates

## Mobile

The Flutter project includes `flutter_test` and standard Flutter linting support.

The backend’s contract and parity tests are especially important because the backend must serve both mobile and dashboard clients without allowing their interpretations of the domain to drift.

---

# Operational Safety

This platform contains several examples of engineering designed specifically to reduce risky production changes.

### Controlled reset

Database handover reset has:

- a check mode,
- an explicit `--execute` mode,
- and dedicated reset-safety tests.

### Production indexes

Indexes are created through an explicit script rather than being treated as an undocumented manual step.

### Audits

Audits exist for:

- active subscriptions,
- pickup addresses,
- add-on conflicts,
- premium eligibility.

### Validation

Validation scripts cover:

- backend state,
- staging,
- data integrity,
- menu identity,
- catalog health.

### Repair/backfill

Repair and migration scripts are named and version-controlled so data maintenance can be reviewed before execution.

---

# Engineering Conventions

Across the repositories, the codebase follows several recurring ideas.

### Keep domain rules authoritative

Business-sensitive rules should live on the backend.

### Keep clients focused

Clients should handle interaction, local validation, display state, and API synchronization.

### Separate operational domains

Accounting, delivery, payments, subscriptions, menu, users, promo codes, and other areas have dedicated routes/workflows rather than being compressed into one generic admin page.

### Make maintenance explicit

Backfills, repairs, bootstrap operations, audits, and migrations should be scripts instead of undocumented one-off production commands.

### Test contracts that cross repositories

When mobile and dashboard consume the same backend, compatibility is part of correctness.

### Treat concurrency as a real risk

Checkout, subscription fulfilment, invoice generation, and entitlement balances can be affected by repeated/concurrent requests, so targeted concurrency and idempotency tests are part of the backend suite.

---

# Documentation Strategy

The organization keeps shared engineering documentation in a separate repository:

[`Basic-Diet/documentations`](https://github.com/Basic-Diet/documentations)

That repository is intended for guidance that crosses application boundaries, such as:

- Git/GitHub workflow
- Shared collaboration rules
- Husky/local automation references
- Documentation conventions
- Cross-repository handoff guidance
- Code-inspection/Graphify references

Application-specific documentation stays in the application repository that owns the code.

This reduces the chance that a central document claims something that no longer matches the live implementation.

---

# Development and Release Model

The platform does not use one universal build command because each application has a different stack.

## Dashboard

Typical verification:

```bash
npm run lint
npm run typecheck
npm run test
npm run build
```

## Backend

The backend has both focused tests and broad release gates.

For high-confidence validation, the repository exposes:

```bash
npm run test:release-gates
```

There are also specialized commands for changed contracts, integration behavior, security, checkout, subscriptions, entitlements, mobile contracts, payments, and catalog behavior.

## Mobile

The mobile app uses the normal Flutter toolchain for dependency resolution, linting, testing, and builds.

---

# Project Status

Basic Diet is an actively maintained production-oriented platform.

The repositories represent a connected operating system rather than isolated demos:

- The **dashboard** supports internal administration and day-to-day operational workflows.
- The **backend** owns authoritative business behavior and persistence.
- The **mobile app** provides the customer experience.
- The **documentation repository** preserves shared engineering workflow and handoff standards.

Individual repositories may continue to evolve as product requirements, operational rules, customer journeys, and backend contracts change.

---

# Repository Guide for Reviewers

If you are reviewing the platform for engineering scope, the most useful order is:

### 1. Start with the dashboard

[`Basic-Diet/client_dashbourd`](https://github.com/Basic-Diet/client_dashbourd)

This gives the fastest view of the operational breadth of the product: subscriptions, customers, menu, accounting, payments, fulfilment, orders, staff, and internal workflows.

### 2. Review the backend

[`Basic-Diet/backend`](https://github.com/Basic-Diet/backend)

This is where the platform’s business complexity becomes most visible. Pay particular attention to the test scripts, release gates, subscription/entitlement rules, catalog contracts, security coverage, concurrency tests, and operational tooling.

### 3. Review the mobile application

[`Basic-Diet/mobile_app`](https://github.com/Basic-Diet/mobile_app)

This shows the customer-facing architecture, Flutter/BLoC structure, localization, secure local storage, routing, and mobile data-layer choices.

### 4. Review shared documentation

[`Basic-Diet/documentations`](https://github.com/Basic-Diet/documentations)

This explains the shared development and collaboration conventions that sit above the individual repositories.

---

## In Short

Basic Diet combines customer-facing mobile software, a complex internal operations dashboard, and a heavily tested backend around the same business domain.

Its engineering challenge is not simply rendering meals or storing users. The harder part is keeping subscriptions, entitlements, orders, fulfilment, payments, catalog rules, roles, and operational actions consistent across multiple clients while preserving a clear security and data-ownership boundary.

That cross-application consistency is the main reason the platform is organized as an ecosystem rather than as a single application repository.
