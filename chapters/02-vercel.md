# Chapter 2: Vercel & Edge Computing

> "The best deployment is the one you don't think about."

---

## What is Vercel?

Vercel is a **frontend cloud platform** that pioneered the modern deployment experience. Founded by Guillermo Rauch (creator of Socket.io and Next.js), Vercel has become the de facto standard for deploying web applications.

```
┌─────────────────────────────────────────────────────────────┐
│                      VERCEL ECOSYSTEM                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   CREATED BY VERCEL         INTEGRATES WITH                 │
│   ─────────────────         ───────────────                 │
│   • Next.js                 • Any Git provider              │
│   • Turbopack               • Any frontend framework        │
│   • SWR                     • Neon, PlanetScale, Supabase   │
│   • Turborepo               • Upstash, Redis Labs           │
│   • AI SDK (Vercel AI)      • Stripe, Auth0, Clerk          │
│                             • And 100+ integrations         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Why Vercel Matters

### The Deploy Experience

```bash
# That's it. That's the deployment.
vercel

# Or just push to Git
git push origin main
# → Vercel automatically deploys
```

What happens behind that simple command:

```
YOUR `git push`
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    VERCEL BUILD PIPELINE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. DETECT FRAMEWORK                                        │
│     └─> Next.js? Nuxt? Astro? SvelteKit? → Auto-configure   │
│                                                              │
│  2. INSTALL DEPENDENCIES                                    │
│     └─> npm/yarn/pnpm install (cached for speed)            │
│                                                              │
│  3. BUILD                                                   │
│     └─> Framework-specific build                            │
│     └─> Static assets optimized                             │
│     └─> Functions bundled                                   │
│                                                              │
│  4. DEPLOY TO EDGE                                          │
│     └─> Static → Global CDN (instant)                       │
│     └─> Functions → Edge or Regional                        │
│     └─> SSL certificate provisioned                         │
│                                                              │
│  5. PREVIEW URL GENERATED                                   │
│     └─> https://your-app-abc123-yourname.vercel.app         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                      │
                      ▼
            Deployed in ~30 seconds
            Available globally
            HTTPS enabled
            Preview comment in PR
```

---

## Vercel's Three Compute Tiers

Understanding where your code runs is crucial:

```
┌─────────────────────────────────────────────────────────────┐
│                    VERCEL COMPUTE TIERS                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. STATIC (CDN)                                       │  │
│  │    ─────────────                                      │  │
│  │    • HTML, CSS, JS, images                            │  │
│  │    • Cached at 100+ edge locations                    │  │
│  │    • Instant response (<50ms globally)                │  │
│  │    • Free tier: Unlimited bandwidth                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. EDGE FUNCTIONS                                     │  │
│  │    ──────────────                                     │  │
│  │    • Run in V8 isolates (like Cloudflare Workers)     │  │
│  │    • ~40 global locations                             │  │
│  │    • Cold start: ~0ms (always warm)                   │  │
│  │    • Max execution: 30 seconds                        │  │
│  │    • Limited APIs (no Node.js native modules)         │  │
│  │    • Best for: Auth, A/B tests, geolocation           │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. SERVERLESS FUNCTIONS (Node.js/Python/Go/Ruby)      │  │
│  │    ────────────────────                               │  │
│  │    • Full Node.js runtime                             │  │
│  │    • Regional (choose: iad1, sfo1, etc.)              │  │
│  │    • Cold start: ~250ms (can be optimized)            │  │
│  │    • Max execution: 10s (Hobby) to 300s (Pro)         │  │
│  │    • Best for: Database queries, heavy computation    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Choosing the Right Tier

```javascript
// EDGE FUNCTION - Runs globally, super fast
// Good for: middleware, auth checks, redirects

// app/middleware.ts (Next.js)
export const config = { matcher: '/dashboard/:path*' };

export function middleware(request) {
  const token = request.cookies.get('session');

  if (!token) {
    return Response.redirect(new URL('/login', request.url));
  }

  // Add user info to headers for downstream use
  const response = NextResponse.next();
  response.headers.set('x-user-id', decodeToken(token).userId);
  return response;
}
```

```javascript
// SERVERLESS FUNCTION - Full Node.js power
// Good for: database operations, third-party APIs

// app/api/users/route.ts (Next.js App Router)
import { db } from '@/lib/db';

export async function GET() {
  // This needs full Node.js - database drivers, etc.
  const users = await db.query('SELECT * FROM users LIMIT 10');

  return Response.json(users);
}

// Optionally specify region close to your database
export const runtime = 'nodejs';
export const preferredRegion = 'iad1'; // US East (close to Neon)
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Rule of thumb:                          │
│ • Need Node.js modules? → Serverless    │
│ • Need global low latency? → Edge       │
│ • Need max performance? → Static/ISR    │
└─────────────────────────────────────────┘
```

---

## Preview Deployments: The Killer Feature

Every pull request gets its own URL:

```
main branch         → your-app.vercel.app
feature/login PR    → your-app-git-feature-login-yourname.vercel.app
fix/button-style PR → your-app-git-fix-button-style-yourname.vercel.app
```

```
┌─────────────────────────────────────────────────────────────┐
│                   PREVIEW DEPLOYMENT FLOW                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Developer                                                  │
│       │                                                      │
│       │ Opens PR                                             │
│       ▼                                                      │
│   ┌─────────────┐                                           │
│   │   GitHub    │──── Webhook ────▶ Vercel starts build     │
│   └─────────────┘                                           │
│       │                                                      │
│       │ ◄─── Vercel posts comment ────                      │
│       │                                                      │
│       ▼                                                      │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  PR #42: Add login page                             │   │
│   │  ─────────────────────────────────────────────────  │   │
│   │                                                     │   │
│   │  ▸ vercel bot                                       │   │
│   │    Preview: https://app-abc123.vercel.app          │   │
│   │    ✓ Build succeeded                                │   │
│   │    Inspect: https://vercel.com/yourname/app/abc123 │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   → QA clicks preview, tests the feature                    │
│   → Designer reviews the UI                                 │
│   → PM approves the functionality                           │
│   → Merge with confidence                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Environment Variables & Secrets

```
┌─────────────────────────────────────────────────────────────┐
│                  ENVIRONMENT MANAGEMENT                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│   │ Development │  │   Preview   │  │ Production  │        │
│   ├─────────────┤  ├─────────────┤  ├─────────────┤        │
│   │ DATABASE_   │  │ DATABASE_   │  │ DATABASE_   │        │
│   │ URL=        │  │ URL=        │  │ URL=        │        │
│   │ local       │  │ staging-db  │  │ prod-db     │        │
│   │             │  │             │  │             │        │
│   │ STRIPE_KEY= │  │ STRIPE_KEY= │  │ STRIPE_KEY= │        │
│   │ sk_test_... │  │ sk_test_... │  │ sk_live_... │        │
│   └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                              │
│   Each environment can have different values                │
│   Secrets are encrypted at rest                             │
│   Never exposed to client-side code (unless prefixed)       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Add secrets via CLI
vercel env add DATABASE_URL production
# Prompts for value, stores encrypted

# Or link to external secret managers
# Vercel integrates with 1Password, HashiCorp Vault, etc.
```

```
┌─────────────────────────────────────────┐
│ ⚠️  WARNING                             │
│                                         │
│ In Next.js:                             │
│ • NEXT_PUBLIC_* → Exposed to browser    │
│ • Everything else → Server-only         │
│                                         │
│ NEVER put secrets in NEXT_PUBLIC_*      │
│ variables. They will be in your         │
│ client-side JavaScript bundle.          │
└─────────────────────────────────────────┘
```

---

## Vercel Integrations Marketplace

One-click setup for common services:

```
┌─────────────────────────────────────────────────────────────┐
│                   POPULAR INTEGRATIONS                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DATABASE                   ANALYTICS                        │
│  ────────                   ─────────                        │
│  • Neon                     • Vercel Analytics              │
│  • PlanetScale              • Vercel Speed Insights         │
│  • Supabase                 • PostHog                       │
│  • MongoDB Atlas            • Amplitude                     │
│                                                              │
│  CACHE/STORAGE              AUTH                            │
│  ────────────               ────                            │
│  • Upstash Redis            • Clerk                         │
│  • Vercel KV                • Auth0                         │
│  • Vercel Blob              • Supabase Auth                 │
│  • Cloudflare R2                                            │
│                                                              │
│  AI/ML                      CMS                             │
│  ─────                      ───                             │
│  • OpenAI                   • Sanity                        │
│  • Replicate                • Contentful                    │
│  • Hugging Face             • Strapi                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Integration flow:
1. Click "Add Integration"
2. Authorize the service
3. Environment variables auto-injected
4. Start using immediately
```

---

## Advanced: Edge Config

Store configuration that needs to be read at the edge with **zero latency**:

```javascript
import { get } from '@vercel/edge-config';

export const config = { runtime: 'edge' };

export default async function middleware(request) {
  // Read from Edge Config - globally replicated, ~0ms read time
  const maintenance = await get('maintenance_mode');

  if (maintenance) {
    return new Response('Site under maintenance', { status: 503 });
  }

  // Check feature flags
  const newCheckout = await get('feature_new_checkout');
  if (newCheckout && request.nextUrl.pathname === '/checkout') {
    return NextResponse.rewrite(new URL('/checkout-v2', request.url));
  }

  return NextResponse.next();
}
```

Use cases:
- Feature flags (instant toggle, no redeploy)
- A/B test configurations
- Maintenance mode switches
- IP blocklists
- Redirect rules

---

## Vercel AI SDK

Build AI-powered applications with streaming support:

```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4-turbo'),
    messages,
  });

  return result.toDataStreamResponse();
}
```

```tsx
// app/chat/page.tsx (Client Component)
'use client';

import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit } = useChat();

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}: {m.content}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

## Pricing Reality Check

```
┌─────────────────────────────────────────────────────────────┐
│                      VERCEL PRICING (2025)                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HOBBY (Free)                                               │
│  ───────────                                                │
│  • Personal projects, non-commercial                        │
│  • 100GB bandwidth/month                                    │
│  • 100GB-hours serverless execution                         │
│  • Unlimited preview deployments                            │
│  • ⚠️  Cannot use for commercial projects                   │
│                                                              │
│  PRO ($20/user/month)                                       │
│  ────────────────────                                       │
│  • Commercial use allowed                                   │
│  • 1TB bandwidth included                                   │
│  • 1000GB-hours serverless                                  │
│  • Team features, password protection                       │
│  • 👍 Best for small teams & startups                       │
│                                                              │
│  ENTERPRISE (Custom)                                        │
│  ──────────────────                                         │
│  • SLA guarantees                                           │
│  • Dedicated support                                        │
│  • Custom limits                                            │
│  • SSO, audit logs                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Start with Hobby for learning.          │
│ Move to Pro when you launch.            │
│ Don't optimize for cost until you       │
│ have real traffic—Vercel's free tier    │
│ is generous for development.            │
└─────────────────────────────────────────┘
```

---

## Hands-On: Deploy Your First App

```
┌─────────────────────────────────────────┐
│ 🎯 EXERCISE                             │
│                                         │
│ Deploy a Next.js app to Vercel:         │
│                                         │
│ 1. Create a new Next.js app:            │
│    npx create-next-app@latest my-app    │
│                                         │
│ 2. cd my-app && code .                  │
│                                         │
│ 3. Make a small change to page.tsx      │
│                                         │
│ 4. Push to GitHub:                      │
│    git init                             │
│    git add .                            │
│    git commit -m "Initial commit"       │
│    gh repo create --public --push       │
│                                         │
│ 5. Go to vercel.com/new                 │
│    Import your repository               │
│    Click Deploy                         │
│                                         │
│ 6. Watch the magic happen!              │
│                                         │
│ Bonus: Create a branch, push it,        │
│ and see the preview deployment.         │
└─────────────────────────────────────────┘
```

---

## Common Gotchas

### 1. Function Size Limits

```
┌─────────────────────────────────────────┐
│ ⚠️  WARNING                             │
│                                         │
│ Serverless functions have size limits:  │
│ • Compressed: 50MB (250MB uncompressed) │
│                                         │
│ If your function is too large:          │
│ • Check for unnecessary dependencies    │
│ • Use dynamic imports                   │
│ • Consider edge functions (smaller)     │
└─────────────────────────────────────────┘
```

### 2. Cold Starts

```javascript
// BAD: Heavy imports at top level
import { PDFDocument } from 'pdf-lib'; // 2MB library
import { createCanvas } from 'canvas';  // Native module

export async function GET() {
  // These are loaded even if not used
}

// GOOD: Dynamic imports when needed
export async function GET(request) {
  const { searchParams } = new URL(request.url);

  if (searchParams.get('format') === 'pdf') {
    const { PDFDocument } = await import('pdf-lib');
    // Use PDFDocument
  }
}
```

### 3. Database Connection Limits

```typescript
// BAD: New connection every request
export async function GET() {
  const client = new Client(process.env.DATABASE_URL);
  await client.connect(); // Cold start = new connection
  const result = await client.query('SELECT 1');
  await client.end();
  return Response.json(result);
}

// GOOD: Connection pooling with Neon's serverless driver
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL);

export async function GET() {
  const result = await sql`SELECT 1`;
  return Response.json(result);
}
```

---

## Key Takeaways

1. **Vercel makes deployment trivial** — `git push` and you're live
2. **Preview deployments change how teams work** — Every PR gets a URL
3. **Choose compute tier wisely** — Edge for speed, Serverless for power
4. **Integrations save time** — One-click setup for databases, auth, etc.
5. **Watch your function size** — Bundle carefully for fast cold starts

---

## What's Next?

Now that you can deploy, let's talk about where your data lives. In Chapter 3, we'll explore **Neon** — the serverless Postgres database that pairs perfectly with Vercel.

---

[← Previous: The Modern Landscape](01-modern-landscape.md) | [Back to Contents](../README.md) | [Next: Neon →](03-neon.md)
