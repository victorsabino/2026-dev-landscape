# Chapter 4: Upstash — Serverless Redis & Kafka

> "Fast data doesn't have to be hard."

---

## What is Upstash?

Upstash provides **serverless Redis and Kafka** — two fundamental building blocks for modern applications. Unlike traditional Redis/Kafka that require server management, Upstash offers:

- **HTTP-based APIs** — Works in Edge functions, serverless, anywhere
- **Pay-per-request pricing** — No idle costs
- **Global replication** — Low latency worldwide
- **Zero ops** — No clusters to manage

```
┌─────────────────────────────────────────────────────────────┐
│                    UPSTASH PRODUCTS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  UPSTASH REDIS                                      │    │
│  │  ───────────────                                    │    │
│  │  • Caching                                          │    │
│  │  • Session storage                                  │    │
│  │  • Rate limiting                                    │    │
│  │  • Real-time leaderboards                          │    │
│  │  • Pub/Sub messaging                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  UPSTASH KAFKA                                      │    │
│  │  ───────────────                                    │    │
│  │  • Event streaming                                  │    │
│  │  • Message queues                                   │    │
│  │  • Log aggregation                                  │    │
│  │  • Data pipelines                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  UPSTASH QSTASH                                     │    │
│  │  ──────────────                                     │    │
│  │  • Background jobs                                  │    │
│  │  • Scheduled tasks                                  │    │
│  │  • Webhook delivery                                 │    │
│  │  • Workflow orchestration                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 1: Upstash Redis

### Why Serverless Redis?

Traditional Redis:
```
┌─────────────────────────────────────────────────────────────┐
│  TRADITIONAL REDIS                                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  You manage:                                                │
│  • EC2/VM instances                                         │
│  • Memory allocation                                        │
│  • Replication setup                                        │
│  • Failover configuration                                   │
│  • Security groups                                          │
│  • Monitoring & alerts                                      │
│  • Backups                                                  │
│                                                              │
│  Cost: $50-200+/month minimum (always running)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Upstash Redis:
```
┌─────────────────────────────────────────────────────────────┐
│  UPSTASH REDIS                                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  You manage:                                                │
│  • Your code                                                │
│  • That's it                                                │
│                                                              │
│  Cost: $0.20 per 100K commands                              │
│        Free tier: 10K commands/day                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Getting Started

```typescript
// Install
// npm install @upstash/redis

// lib/redis.ts
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Or use the auto-configured client (reads from env automatically)
// import { Redis } from '@upstash/redis';
// export const redis = Redis.fromEnv();
```

### Common Use Cases

#### 1. Caching

```typescript
// app/api/products/[id]/route.ts
import { redis } from '@/lib/redis';
import { db } from '@/lib/db';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const cacheKey = `product:${params.id}`;

  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return Response.json(cached);
  }

  // Cache miss - fetch from database
  const product = await db.query.products.findFirst({
    where: eq(products.id, parseInt(params.id)),
  });

  if (!product) {
    return Response.json({ error: 'Not found' }, { status: 404 });
  }

  // Cache for 1 hour
  await redis.set(cacheKey, product, { ex: 3600 });

  return Response.json(product);
}
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Cache invalidation patterns:            │
│                                         │
│ 1. Time-based (TTL):                    │
│    redis.set(key, value, { ex: 3600 })  │
│                                         │
│ 2. Event-based:                         │
│    When data changes, delete cache key  │
│                                         │
│ 3. Write-through:                       │
│    Update cache when writing to DB      │
│                                         │
│ Start simple with TTL, add complexity   │
│ only when needed.                       │
└─────────────────────────────────────────┘
```

#### 2. Rate Limiting

```typescript
// lib/rate-limit.ts
import { redis } from './redis';
import { Ratelimit } from '@upstash/ratelimit';

// Create a rate limiter: 10 requests per 10 seconds
export const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '10 s'),
  analytics: true, // Track analytics in Upstash console
});

// app/api/expensive-operation/route.ts
import { ratelimit } from '@/lib/rate-limit';

export async function POST(request: Request) {
  // Use IP or user ID as identifier
  const ip = request.headers.get('x-forwarded-for') ?? 'anonymous';

  const { success, limit, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return Response.json(
      { error: 'Too many requests' },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
        },
      }
    );
  }

  // Process the request
  return Response.json({ success: true });
}
```

#### 3. Session Storage

```typescript
// lib/session.ts
import { redis } from './redis';
import { nanoid } from 'nanoid';

interface Session {
  userId: string;
  email: string;
  createdAt: number;
}

export async function createSession(userId: string, email: string) {
  const sessionId = nanoid();
  const session: Session = {
    userId,
    email,
    createdAt: Date.now(),
  };

  // Store session for 7 days
  await redis.set(`session:${sessionId}`, session, { ex: 60 * 60 * 24 * 7 });

  return sessionId;
}

export async function getSession(sessionId: string): Promise<Session | null> {
  return redis.get(`session:${sessionId}`);
}

export async function destroySession(sessionId: string) {
  await redis.del(`session:${sessionId}`);
}
```

#### 4. Real-time Leaderboards

```typescript
// lib/leaderboard.ts
import { redis } from './redis';

const LEADERBOARD_KEY = 'game:leaderboard';

export async function addScore(userId: string, score: number) {
  // ZADD adds to sorted set (higher score = higher rank)
  await redis.zadd(LEADERBOARD_KEY, { score, member: userId });
}

export async function getTopPlayers(limit = 10) {
  // ZREVRANGE returns highest scores first
  return redis.zrange(LEADERBOARD_KEY, 0, limit - 1, {
    rev: true,
    withScores: true,
  });
}

export async function getPlayerRank(userId: string) {
  // ZREVRANK returns rank (0-indexed, highest first)
  const rank = await redis.zrevrank(LEADERBOARD_KEY, userId);
  return rank !== null ? rank + 1 : null;
}

export async function getPlayerScore(userId: string) {
  return redis.zscore(LEADERBOARD_KEY, userId);
}
```

---

## Part 2: Upstash QStash

QStash is a **serverless message queue** designed for serverless environments. It solves a fundamental problem: serverless functions have time limits, but some tasks take longer.

```
┌─────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   User clicks "Generate Report"                             │
│          │                                                   │
│          ▼                                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Vercel Serverless Function                         │   │
│   │                                                     │   │
│   │  • Fetch data from database                        │   │
│   │  • Process 10,000 rows                             │   │
│   │  • Generate PDF                                     │   │
│   │  • Upload to S3                                     │   │
│   │  • Send email notification                          │   │
│   │                                                     │   │
│   │  Time needed: 45 seconds                            │   │
│   │  Vercel limit: 10 seconds (hobby) / 60s (pro)       │   │
│   │                                                     │   │
│   │  💥 TIMEOUT                                         │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                    THE SOLUTION: QSTASH                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   User clicks "Generate Report"                             │
│          │                                                   │
│          ▼                                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  API Route (fast)                                   │   │
│   │                                                     │   │
│   │  1. Validate request                                │   │
│   │  2. Queue job to QStash                             │   │
│   │  3. Return immediately: "Processing started"        │   │
│   │                                                     │   │
│   │  Time: 200ms ✓                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│          │                                                   │
│          │ QStash calls your endpoint                       │
│          ▼                                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Background Worker Route                             │   │
│   │                                                     │   │
│   │  • Processes in chunks                              │   │
│   │  • QStash retries on failure                        │   │
│   │  • Can take up to 2 hours                          │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### QStash Basics

```typescript
// lib/qstash.ts
import { Client } from '@upstash/qstash';

export const qstash = new Client({
  token: process.env.QSTASH_TOKEN!,
});
```

```typescript
// app/api/reports/route.ts
import { qstash } from '@/lib/qstash';

export async function POST(request: Request) {
  const { reportType, userId } = await request.json();

  // Queue the job - returns immediately
  const response = await qstash.publishJSON({
    url: `${process.env.VERCEL_URL}/api/workers/generate-report`,
    body: { reportType, userId },
    retries: 3, // Retry up to 3 times if it fails
  });

  return Response.json({
    message: 'Report generation started',
    jobId: response.messageId,
  });
}
```

```typescript
// app/api/workers/generate-report/route.ts
import { verifySignatureAppRouter } from '@upstash/qstash/nextjs';

async function handler(request: Request) {
  const { reportType, userId } = await request.json();

  // This can take as long as needed
  // QStash handles retries if it fails
  await generateReport(reportType, userId);
  await uploadToS3();
  await sendEmailNotification(userId);

  return Response.json({ success: true });
}

// Verify the request came from QStash (security)
export const POST = verifySignatureAppRouter(handler);
```

### Scheduled Jobs (Cron)

```typescript
// Create a scheduled job that runs every hour
await qstash.schedules.create({
  destination: `${process.env.VERCEL_URL}/api/cron/cleanup`,
  cron: '0 * * * *', // Every hour
});

// Or use Vercel's cron syntax in vercel.json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/cleanup",
      "schedule": "0 * * * *"
    }
  ]
}
```

### Delayed Messages

```typescript
// Send a reminder email in 24 hours
await qstash.publishJSON({
  url: `${process.env.VERCEL_URL}/api/workers/send-reminder`,
  body: { userId, type: 'trial-expiring' },
  delay: 60 * 60 * 24, // 24 hours in seconds
});
```

---

## Part 3: Upstash Kafka (Overview)

For high-throughput event streaming:

```typescript
// lib/kafka.ts
import { Kafka } from '@upstash/kafka';

export const kafka = new Kafka({
  url: process.env.UPSTASH_KAFKA_REST_URL!,
  username: process.env.UPSTASH_KAFKA_REST_USERNAME!,
  password: process.env.UPSTASH_KAFKA_REST_PASSWORD!,
});
```

```typescript
// Produce events
const producer = kafka.producer();

await producer.produce('user-events', {
  userId: '123',
  action: 'purchase',
  amount: 99.99,
  timestamp: Date.now(),
});

// Consume events
const consumer = kafka.consumer();
const messages = await consumer.consume({
  consumerGroupId: 'analytics-group',
  topics: ['user-events'],
  autoOffsetReset: 'earliest',
});
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ When to use Kafka vs QStash:            │
│                                         │
│ QStash:                                 │
│ • Background jobs                       │
│ • Scheduled tasks                       │
│ • Simple queue patterns                 │
│ • Webhook delivery                      │
│                                         │
│ Kafka:                                  │
│ • High-throughput streaming             │
│ • Event sourcing                        │
│ • Multiple consumers per message        │
│ • Real-time analytics pipelines         │
│                                         │
│ Start with QStash—it's simpler.         │
│ Add Kafka when you need streaming.      │
└─────────────────────────────────────────┘
```

---

## Full Example: API with Caching and Rate Limiting

```typescript
// app/api/posts/[id]/route.ts
import { redis } from '@/lib/redis';
import { ratelimit } from '@/lib/rate-limit';
import { db } from '@/lib/db';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  // 1. Rate limiting
  const ip = request.headers.get('x-forwarded-for') ?? 'anonymous';
  const { success } = await ratelimit.limit(ip);

  if (!success) {
    return Response.json({ error: 'Rate limited' }, { status: 429 });
  }

  // 2. Check cache
  const cacheKey = `post:${params.id}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return Response.json(cached, {
      headers: { 'X-Cache': 'HIT' },
    });
  }

  // 3. Fetch from database
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, parseInt(params.id)),
    with: {
      author: true,
      comments: { limit: 10 },
    },
  });

  if (!post) {
    return Response.json({ error: 'Not found' }, { status: 404 });
  }

  // 4. Cache for 5 minutes
  await redis.set(cacheKey, post, { ex: 300 });

  return Response.json(post, {
    headers: { 'X-Cache': 'MISS' },
  });
}

export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  const body = await request.json();

  // Update database
  const post = await db
    .update(posts)
    .set({ title: body.title, content: body.content })
    .where(eq(posts.id, parseInt(params.id)))
    .returning();

  // Invalidate cache
  await redis.del(`post:${params.id}`);

  return Response.json(post);
}
```

---

## Pricing Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    UPSTASH PRICING (2025)                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REDIS                                                      │
│  ─────                                                      │
│  Free:    10K commands/day, 256MB storage                   │
│  Pay-go:  $0.20 per 100K commands                           │
│  Pro:     $280/month (Global, 10GB, unlimited)              │
│                                                              │
│  QSTASH                                                     │
│  ──────                                                     │
│  Free:    500 messages/day                                  │
│  Pay-go:  $1 per 100K messages                              │
│  Pro:     $180/month (unlimited)                            │
│                                                              │
│  KAFKA                                                      │
│  ─────                                                      │
│  Free:    10K messages/day                                  │
│  Pay-go:  $0.60 per 100K messages                           │
│  Pro:     Starting at $320/month                            │
│                                                              │
│  Note: Free tiers reset daily, perfect for development      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Exercise: Build a Rate-Limited API

```
┌─────────────────────────────────────────┐
│ 🎯 EXERCISE                             │
│                                         │
│ Create an API with:                     │
│                                         │
│ 1. Sign up at upstash.com               │
│                                         │
│ 2. Create a Redis database              │
│                                         │
│ 3. Install dependencies:                │
│    npm install @upstash/redis           │
│    npm install @upstash/ratelimit       │
│                                         │
│ 4. Create lib/redis.ts with client      │
│                                         │
│ 5. Create lib/rate-limit.ts with:       │
│    • 5 requests per minute per IP       │
│                                         │
│ 6. Create API route /api/test           │
│    • Check rate limit                   │
│    • Return 429 if exceeded             │
│    • Return success otherwise           │
│                                         │
│ 7. Test by hitting endpoint rapidly     │
│    curl -X GET localhost:3000/api/test  │
│                                         │
│ BONUS: Add caching to a database query  │
└─────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Redis is essential** — Caching, sessions, rate limiting, leaderboards
2. **HTTP-based = serverless-friendly** — Works anywhere, even Edge
3. **QStash solves background jobs** — Essential for long-running tasks
4. **Pay-per-request is economical** — Free tier covers development
5. **Start simple** — Redis caching + rate limiting cover 80% of needs

---

## What's Next?

Now that you understand the infrastructure layer (Vercel, Neon, Upstash), it's time to dive into the code itself. In Chapter 5, we'll explore why **Fastify is replacing Express** as the go-to Node.js framework.

---

[← Previous: Neon](03-neon.md) | [Back to Contents](../README.md) | [Next: Express to Fastify →](05-express-to-fastify.md)
