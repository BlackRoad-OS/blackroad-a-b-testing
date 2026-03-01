<div align="center">

# 🛣️ 🧪 @blackroad/a-b-testing

**Production-grade A/B testing for the modern web.**

Statistically rigorous experimentation — consistent hashing, chi-squared significance, Stripe-native checkout variants, and first-class TypeScript support.

[![npm version](https://img.shields.io/npm/v/@blackroad/a-b-testing?style=flat-square&color=FF1D6C)](https://www.npmjs.com/package/@blackroad/a-b-testing)
[![npm downloads](https://img.shields.io/npm/dm/@blackroad/a-b-testing?style=flat-square)](https://www.npmjs.com/package/@blackroad/a-b-testing)
[![License](https://img.shields.io/badge/license-Proprietary-9C27B0?style=flat-square)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square)](https://www.typescriptlang.org/)
[![Platform](https://img.shields.io/badge/Platform-blackroad.io-FF1D6C?style=flat-square)](https://blackroad.io)

</div>

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Requirements](#requirements)
4. [Installation](#installation)
5. [Quick Start](#quick-start)
6. [Core Concepts](#core-concepts)
   - [Experiments](#experiments)
   - [Variants](#variants)
   - [Consistent Hashing](#consistent-hashing)
   - [Statistical Significance](#statistical-significance)
7. [API Reference](#api-reference)
   - [ABTestingClient](#abtestingclient)
   - [createExperiment](#createexperiment)
   - [assignVariant](#assignvariant)
   - [recordConversion](#recordconversion)
   - [getSignificance](#getsignificance)
   - [getWinner](#getwinner)
8. [Stripe Integration](#stripe-integration)
   - [A/B Testing Stripe Pricing](#ab-testing-stripe-pricing)
   - [Checkout Variant Experiments](#checkout-variant-experiments)
   - [Subscription Plan Experiments](#subscription-plan-experiments)
   - [Webhook Handling](#webhook-handling)
9. [Framework Integrations](#framework-integrations)
   - [Next.js](#nextjs)
   - [React](#react)
   - [Node.js / Express](#nodejs--express)
   - [Cloudflare Workers](#cloudflare-workers)
10. [End-to-End Example](#end-to-end-example)
11. [Configuration Reference](#configuration-reference)
12. [TypeScript Types](#typescript-types)
13. [Statistical Methodology](#statistical-methodology)
14. [Security](#security)
15. [Changelog](#changelog)
16. [License](#license)
17. [Support](#support)

---

## Overview

`@blackroad/a-b-testing` is the official A/B testing library from [BlackRoad OS](https://blackroad.io) — the operating system for AI-powered applications. It provides everything you need to run rigorous experiments on any user-facing surface, with purpose-built support for **Stripe** payment and subscription flows.

Built for teams that need deterministic assignment, real statistical confidence, and zero external service dependencies for core experiment logic.

---

## Features

| Feature | Description |
|---|---|
| 🔐 **Consistent hashing** | Users always see the same variant across sessions and devices |
| 📊 **Chi-squared significance** | Know exactly when your results are statistically valid |
| 💳 **Stripe-native integration** | A/B test prices, checkout sessions, and subscription plans |
| 🚦 **Multi-variant support** | Run A/B/C/D/… experiments with arbitrary traffic splits |
| 🏎️ **Edge-ready** | Runs in Cloudflare Workers, Vercel Edge, and Deno |
| 📦 **Zero peer dependencies** | Core library has no required third-party dependencies |
| 🔒 **Type-safe** | First-class TypeScript with full type inference |
| 🪝 **Framework adapters** | Drop-in support for Next.js, React, and Express |
| 📡 **Webhook-aware** | Record Stripe conversion events automatically |
| 🧪 **Testable** | Deterministic assignments make unit testing trivial |

---

## Requirements

- **Node.js** ≥ 18.0.0 (or any runtime with the Web Crypto API)
- **npm** ≥ 9.0.0 (or pnpm / yarn / bun)
- **TypeScript** ≥ 5.0 *(optional but recommended)*
- **Stripe** account *(only required for Stripe integration features)*

---

## Installation

### npm

```bash
npm install @blackroad/a-b-testing
```

### pnpm

```bash
pnpm add @blackroad/a-b-testing
```

### yarn

```bash
yarn add @blackroad/a-b-testing
```

### bun

```bash
bun add @blackroad/a-b-testing
```

---

## Quick Start

```typescript
import { ABTestingClient } from '@blackroad/a-b-testing';

const client = new ABTestingClient();

// 1. Create an experiment
await client.createExperiment({
  name: 'cta-button-color',
  description: 'Test whether a red or blue CTA increases sign-ups',
  variants: [
    { name: 'control',   trafficPct: 50 },
    { name: 'treatment', trafficPct: 50 },
  ],
  metric: 'signup',
});

// 2. Assign a user deterministically
const variant = await client.assignVariant('cta-button-color', { userId: 'user-123' });
console.log(variant.name); // 'control' | 'treatment'

// 3. Record a conversion
await client.recordConversion('cta-button-color', { userId: 'user-123' });

// 4. Check results
const results = await client.getSignificance('cta-button-color', { confidence: 0.95 });
console.log(results.isSignificant);   // true
console.log(results.recommendedWinner); // 'treatment'
```

---

## Core Concepts

### Experiments

An **experiment** defines the scope of a test. Every experiment has:

- A unique `name` used as the hashing seed
- One or more `variants` with traffic percentage allocations that sum to 100
- An optional `metric` name that conversions are tracked against
- A lifecycle: `draft → running → stopped`

### Variants

A **variant** is a distinct version of a feature being tested. Traffic is allocated by percentage. The `control` is conventionally the existing experience; all other variants are treatments.

```typescript
const variants = [
  { name: 'control',     trafficPct: 33.3 },
  { name: 'treatment-a', trafficPct: 33.3 },
  { name: 'treatment-b', trafficPct: 33.4 },
];
```

### Consistent Hashing

Assignment uses SHA-256 hashing on `${experimentName}:${userId}` to produce a stable bucket in `[0, 10000)`. Variants are mapped to contiguous bucket ranges proportional to their traffic percentages.

**Guarantees:**
- The same user always receives the same variant for a given experiment
- Traffic is uniformly distributed across the hash space
- Renaming an experiment produces independent assignments

### Statistical Significance

Results are evaluated using the **chi-squared (χ²) test of independence** on a 2 × n contingency table of assignments and conversions. The library compares the computed χ² statistic against critical values at the requested confidence level (90 %, 95 %, 99 %, 99.9 %).

---

## API Reference

### `ABTestingClient`

```typescript
import { ABTestingClient } from '@blackroad/a-b-testing';

const client = new ABTestingClient(options?: ABTestingClientOptions);
```

| Option | Type | Default | Description |
|---|---|---|---|
| `storage` | `StorageAdapter` | `InMemoryAdapter` | Persistence layer |
| `hashSalt` | `string` | `''` | Additional secret salt for hashing |
| `defaultConfidence` | `number` | `0.95` | Default confidence level for significance tests |

---

### `createExperiment`

```typescript
await client.createExperiment(experiment: ExperimentConfig): Promise<Experiment>
```

Creates and persists a new experiment in `draft` status. Call `startExperiment` to begin assigning users.

```typescript
await client.createExperiment({
  name: 'pricing-page-layout',
  description: 'Test a simplified pricing layout',
  hypothesis: 'Simplified layout increases plan upgrades by 10 %',
  variants: [
    { name: 'control',   trafficPct: 50, description: 'Current layout' },
    { name: 'treatment', trafficPct: 50, description: 'Simplified layout' },
  ],
  metric: 'upgrade',
  tags: ['pricing', 'growth'],
});
```

---

### `assignVariant`

```typescript
await client.assignVariant(
  experimentName: string,
  context: AssignmentContext
): Promise<Variant>
```

Returns the deterministic variant for the given user. Safe to call on every request — assignments are idempotent.

```typescript
const variant = await client.assignVariant('pricing-page-layout', {
  userId: req.user.id,
  attributes: { plan: req.user.plan, country: req.geo.country },
});
```

---

### `recordConversion`

```typescript
await client.recordConversion(
  experimentName: string,
  context: ConversionContext
): Promise<void>
```

Records a conversion event. Duplicate conversions for the same user and experiment are ignored (idempotent).

```typescript
await client.recordConversion('pricing-page-layout', {
  userId: req.user.id,
  value: 49.00,           // Optional — monetary value of the conversion
  metadata: { plan: 'pro' },
});
```

---

### `getSignificance`

```typescript
await client.getSignificance(
  experimentName: string,
  options?: SignificanceOptions
): Promise<SignificanceResult>
```

```typescript
const result = await client.getSignificance('pricing-page-layout', {
  confidence: 0.95,
});

// SignificanceResult
{
  variantStats: {
    control:   { assignments: 1024, conversions: 82,  conversionRatePct: 8.01 },
    treatment: { assignments: 976,  conversions: 147, conversionRatePct: 15.06 },
  },
  chi2Stat: 28.74,
  pValue: 0.00001,
  isSignificant: true,
  recommendedWinner: 'treatment',
  reason: 'Chi2=28.74 > critical=3.841 at 95% confidence (p≈0.00001)',
}
```

---

### `getWinner`

```typescript
await client.getWinner(experimentName: string): Promise<string | null>
```

Returns the name of the winning variant if the experiment is statistically significant at the configured confidence level, otherwise `null`.

---

## Stripe Integration

`@blackroad/a-b-testing` ships a first-class Stripe adapter that lets you A/B test prices, checkout sessions, and subscription plans without any custom event plumbing.

### Installation

```bash
npm install @blackroad/a-b-testing stripe
```

### A/B Testing Stripe Pricing

Test whether a **monthly/annual toggle** or different **price points** affect conversions:

```typescript
import Stripe from 'stripe';
import { ABTestingClient, StripeAdapter } from '@blackroad/a-b-testing';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const client = new ABTestingClient();

const stripeAdapter = new StripeAdapter({ stripe, client });

// Create a pricing experiment
await client.createExperiment({
  name: 'pro-plan-price',
  variants: [
    { name: 'control',   trafficPct: 50 }, // $29/month
    { name: 'treatment', trafficPct: 50 }, // $24/month
  ],
  metric: 'stripe_subscription_created',
});

// In your API route / server action:
const variant = await client.assignVariant('pro-plan-price', {
  userId: user.id,
});

const priceId = variant.name === 'treatment'
  ? process.env.STRIPE_PRICE_24_USD   // discounted price
  : process.env.STRIPE_PRICE_29_USD;  // standard price
```

---

### Checkout Variant Experiments

Test **checkout UX variations** (e.g. number of form fields, payment method ordering):

```typescript
import { ABTestingClient, StripeAdapter } from '@blackroad/a-b-testing';

const variant = await client.assignVariant('checkout-flow', {
  userId: session.userId,
});

// Create a Stripe Checkout Session scoped to the variant
const checkoutSession = await stripe.checkout.sessions.create({
  line_items: [{ price: priceId, quantity: 1 }],
  mode: 'subscription',
  success_url: `${process.env.APP_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${process.env.APP_URL}/pricing`,
  metadata: {
    ab_experiment: 'checkout-flow',
    ab_variant:    variant.name,
    ab_user_id:    session.userId,
  },
  // Variant-specific options
  payment_method_types: variant.name === 'treatment'
    ? ['card', 'link']   // Express Checkout enabled
    : ['card'],
});
```

---

### Subscription Plan Experiments

Record a Stripe subscription creation as a conversion automatically:

```typescript
const stripeAdapter = new StripeAdapter({ stripe, client });

// Pass raw Stripe webhook events — the adapter maps them automatically
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const event = stripe.webhooks.constructEvent(
    req.body,
    req.headers['stripe-signature']!,
    process.env.STRIPE_WEBHOOK_SECRET!,
  );

  await stripeAdapter.handleWebhookEvent(event);
  res.json({ received: true });
});
```

`StripeAdapter.handleWebhookEvent` automatically records conversions for:

| Stripe Event | Mapped Metric |
|---|---|
| `checkout.session.completed` | `stripe_checkout_completed` |
| `customer.subscription.created` | `stripe_subscription_created` |
| `customer.subscription.updated` | `stripe_subscription_updated` |
| `invoice.paid` | `stripe_invoice_paid` |
| `payment_intent.succeeded` | `stripe_payment_succeeded` |

---

### Webhook Handling

```typescript
import { StripeAdapter } from '@blackroad/a-b-testing';

const adapter = new StripeAdapter({
  stripe,
  client,
  // Optional: override which events map to conversions
  conversionEvents: {
    'checkout.session.completed': 'checkout_completed',
    'customer.subscription.created': 'subscription_created',
  },
});
```

---

## Framework Integrations

### Next.js

```typescript
// app/pricing/page.tsx
import { ABTestingClient } from '@blackroad/a-b-testing';
import { cookies } from 'next/headers';

const client = new ABTestingClient();

export default async function PricingPage() {
  const cookieStore = await cookies();
  const userId = cookieStore.get('userId')?.value ?? crypto.randomUUID();

  const variant = await client.assignVariant('pricing-layout', { userId });

  return variant.name === 'treatment'
    ? <NewPricingLayout />
    : <CurrentPricingLayout />;
}
```

```typescript
// app/api/convert/route.ts
import { ABTestingClient } from '@blackroad/a-b-testing';
import { NextRequest, NextResponse } from 'next/server';

const client = new ABTestingClient();

export async function POST(req: NextRequest) {
  const { experimentName, userId, value } = await req.json();
  await client.recordConversion(experimentName, { userId, value });
  return NextResponse.json({ ok: true });
}
```

---

### React

```typescript
// hooks/useVariant.ts
import { useEffect, useState } from 'react';

export function useVariant(experimentName: string, userId: string) {
  const [variant, setVariant] = useState<string | null>(null);

  useEffect(() => {
    fetch('/api/ab/assign', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ experimentName, userId }),
    })
      .then(r => r.json())
      .then(data => setVariant(data.variant));
  }, [experimentName, userId]);

  return variant;
}

// In a component
function PricingButton({ userId }: { userId: string }) {
  const variant = useVariant('cta-color', userId);

  return (
    <button style={{ background: variant === 'treatment' ? '#FF1D6C' : '#1a1a1a' }}>
      Get Started
    </button>
  );
}
```

---

### Node.js / Express

```typescript
import express from 'express';
import { ABTestingClient } from '@blackroad/a-b-testing';

const app = express();
const client = new ABTestingClient();

app.get('/pricing', async (req, res) => {
  const userId = req.session.userId;
  const variant = await client.assignVariant('pricing-layout', { userId });

  res.render('pricing', { layout: variant.name });
});

app.post('/purchase', async (req, res) => {
  const { userId } = req.session;
  await client.recordConversion('pricing-layout', { userId, value: req.body.amount });
  res.json({ ok: true });
});
```

---

### Cloudflare Workers

```typescript
import { ABTestingClient, CloudflareKVAdapter } from '@blackroad/a-b-testing';

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const client = new ABTestingClient({
      storage: new CloudflareKVAdapter(env.AB_TESTING_KV),
    });

    const userId = new URL(request.url).searchParams.get('uid') ?? 'anonymous';
    const variant = await client.assignVariant('edge-test', { userId });

    return new Response(JSON.stringify({ variant: variant.name }), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

---

## End-to-End Example

The following example runs a complete pricing experiment with Stripe on a Next.js application.

### 1. Configure environment variables

```bash
# .env.local
STRIPE_SECRET_KEY=sk_live_…
STRIPE_WEBHOOK_SECRET=whsec_…
STRIPE_PRICE_CONTROL=price_…      # $29/month
STRIPE_PRICE_TREATMENT=price_…    # $24/month
```

### 2. Initialize the experiment (run once at startup or via a script)

```typescript
// scripts/seed-experiments.ts
import { ABTestingClient } from '@blackroad/a-b-testing';

const client = new ABTestingClient();

await client.createExperiment({
  name: 'pro-pricing-2026-q1',
  description: 'Test $24 vs $29 monthly Pro plan',
  hypothesis: 'Lower price increases monthly conversion by ≥ 15 %',
  variants: [
    { name: 'control',   trafficPct: 50, description: '$29/month' },
    { name: 'treatment', trafficPct: 50, description: '$24/month' },
  ],
  metric: 'stripe_subscription_created',
});

await client.startExperiment('pro-pricing-2026-q1');
console.log('Experiment started.');
```

### 3. Assign variant on the pricing page

```typescript
// app/pricing/page.tsx
import { ABTestingClient } from '@blackroad/a-b-testing';
import { cookies } from 'next/headers';

const client = new ABTestingClient();

export default async function PricingPage() {
  const cookieStore = await cookies();
  const userId = cookieStore.get('br_uid')?.value;

  const variant = await client.assignVariant('pro-pricing-2026-q1', { userId });
  const priceId = variant.name === 'treatment'
    ? process.env.STRIPE_PRICE_TREATMENT
    : process.env.STRIPE_PRICE_CONTROL;

  return <PricingCard priceId={priceId} variantName={variant.name} />;
}
```

### 4. Create the Stripe Checkout Session

```typescript
// app/api/checkout/route.ts
import Stripe from 'stripe';
import { NextRequest, NextResponse } from 'next/server';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  const { priceId, userId, variantName } = await req.json();

  const session = await stripe.checkout.sessions.create({
    mode: 'subscription',
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${process.env.APP_URL}/dashboard?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.APP_URL}/pricing`,
    metadata: {
      ab_experiment: 'pro-pricing-2026-q1',
      ab_variant:    variantName,
      ab_user_id:    userId,
    },
  });

  return NextResponse.json({ url: session.url });
}
```

### 5. Record conversions via Stripe webhook

```typescript
// app/api/webhooks/stripe/route.ts
import Stripe from 'stripe';
import { ABTestingClient, StripeAdapter } from '@blackroad/a-b-testing';
import { NextRequest, NextResponse } from 'next/server';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const client = new ABTestingClient();
const adapter = new StripeAdapter({ stripe, client });

export async function POST(req: NextRequest) {
  const body = await req.text();
  const sig  = req.headers.get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }

  await adapter.handleWebhookEvent(event);
  return NextResponse.json({ received: true });
}
```

### 6. Review results

```typescript
// scripts/check-results.ts
import { ABTestingClient } from '@blackroad/a-b-testing';

const client = new ABTestingClient();
const result = await client.getSignificance('pro-pricing-2026-q1', { confidence: 0.95 });

console.log(JSON.stringify(result, null, 2));
```

```json
{
  "variantStats": {
    "control":   { "assignments": 2048, "conversions": 164, "conversionRatePct": 8.01 },
    "treatment": { "assignments": 1952, "conversions": 294, "conversionRatePct": 15.06 }
  },
  "chi2Stat": 72.18,
  "pValue": 0.000000001,
  "isSignificant": true,
  "recommendedWinner": "treatment",
  "reason": "Chi2=72.18 > critical=3.841 at 95% confidence (p≈0.000000001)"
}
```

---

## Configuration Reference

```typescript
interface ABTestingClientOptions {
  /**
   * Storage adapter for experiment and assignment persistence.
   * @default InMemoryAdapter (use a persistent adapter in production)
   */
  storage?: StorageAdapter;

  /**
   * Secret salt mixed into the hashing function to prevent external
   * enumeration of variant assignments.
   * @default ''
   */
  hashSalt?: string;

  /**
   * Default statistical confidence level for significance tests.
   * Supported: 0.90 | 0.95 | 0.99 | 0.999
   * @default 0.95
   */
  defaultConfidence?: 0.90 | 0.95 | 0.99 | 0.999;

  /**
   * Logger instance (must implement { info, warn, error }).
   * @default console
   */
  logger?: Logger;
}
```

---

## TypeScript Types

```typescript
import type {
  ABTestingClient,
  ABTestingClientOptions,
  Experiment,
  ExperimentConfig,
  ExperimentStatus,
  Variant,
  VariantConfig,
  AssignmentContext,
  ConversionContext,
  SignificanceOptions,
  SignificanceResult,
  VariantStats,
  StorageAdapter,
  StripeAdapter,
  StripeAdapterOptions,
} from '@blackroad/a-b-testing';
```

---

## Statistical Methodology

### Chi-Squared Test of Independence

The library evaluates experiment results using the **χ² test of independence** applied to the 2 × n contingency table of (assignments, conversions) per variant:

|            | Converted  | Not Converted  |
|------------|-----------|----------------|
| Variant 1  | c₁        | n₁ − c₁        |
| Variant 2  | c₂        | n₂ − c₂        |
| …          | …         | …              |

```
χ² = Σᵢ (Oᵢ − Eᵢ)² / Eᵢ
```

where expected frequencies **Eᵢ** are computed from row and column marginals, and degrees of freedom = `n_variants − 1`.

### Confidence Levels

| Confidence | α     | Critical Value (df=1) |
|-----------|-------|-----------------------|
| 90 %      | 0.10  | 2.706                 |
| **95 %**  | **0.05** | **3.841**          |
| 99 %      | 0.01  | 6.635                 |
| 99.9 %    | 0.001 | 10.828                |

### Consistent Hashing

```
bucket = (SHA256("${experimentName}:${hashSalt}:${userId}") mod 10,000) / 100
```

Variants occupy contiguous ranges of `[0, 100)` proportional to their `trafficPct`. This guarantees stable, uniform assignments.

---

## Security

- **Never expose `STRIPE_SECRET_KEY` or `STRIPE_WEBHOOK_SECRET` client-side.** Use environment variables exclusively.
- **Validate all Stripe webhook events** using `stripe.webhooks.constructEvent` before processing. The `StripeAdapter` handles this automatically when you pass in events already verified by your webhook handler.
- **Set `hashSalt`** in production to prevent external enumeration of experiment assignments.
- Proprietary license — see [LICENSE](./LICENSE) for permitted use.

---

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for release history.

---

## License

**Proprietary** — © BlackRoad OS, Inc. All rights reserved.

This software is licensed under the [BlackRoad OS Proprietary License](./LICENSE). Unauthorized use, reproduction, or distribution is prohibited. Contact [legal@blackroad.io](mailto:legal@blackroad.io) for licensing inquiries.

---

## Support

| Channel | Link |
|---|---|
| 📖 Documentation | [docs.blackroad.io](https://docs.blackroad.io) |
| 🌐 Website | [blackroad.io](https://blackroad.io) |
| 💬 Status | [status.blackroad.io](https://status.blackroad.io) |
| 📧 Enterprise Support | [support@blackroad.io](mailto:support@blackroad.io) |
| 🔐 Security | [security@blackroad.io](mailto:security@blackroad.io) |

---

<div align="center">

**Part of [BlackRoad OS](https://blackroad.io)** — The Operating System for AI

*© BlackRoad OS, Inc. All rights reserved.*

</div>
