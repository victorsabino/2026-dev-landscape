# Chapter 3: Neon — Serverless Postgres

> "The best database is one that just works."

---

## What is Neon?

Neon is a **serverless Postgres** platform that separates compute from storage. This architectural choice enables features that were previously impossible with traditional Postgres:

- **Scale to zero** — No traffic? No charges.
- **Instant branching** — Copy your entire database in milliseconds
- **Autoscaling** — Handle traffic spikes automatically
- **Point-in-time recovery** — Go back to any moment in time

```
┌─────────────────────────────────────────────────────────────┐
│                    NEON ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   YOUR APPLICATION                                           │
│        │                                                     │
│        │ Connection (pooled)                                 │
│        ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    NEON PROXY                        │   │
│   │  • Connection pooling (PgBouncer built-in)          │   │
│   │  • Route to correct compute                         │   │
│   │  • Handle auth & SSL                                │   │
│   └─────────────────────────────────────────────────────┘   │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    COMPUTE                           │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│   │  │ main    │  │ preview │  │ dev     │  branches   │   │
│   │  │ (0.25-4 │  │ (auto   │  │ (scale  │             │   │
│   │  │  vCPU)  │  │ suspend)│  │ to zero)│             │   │
│   │  └─────────┘  └─────────┘  └─────────┘             │   │
│   └─────────────────────────────────────────────────────┘   │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    STORAGE (Shared)                  │   │
│   │  • Copy-on-write (branching is instant)             │   │
│   │  • Point-in-time recovery (30 days default)         │   │
│   │  • Automatic backups                                 │   │
│   │  • Multi-region replication (Pro)                   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Why Neon Over Traditional Postgres?

### The Serverless Problem with Traditional Databases

```
TRADITIONAL POSTGRES + SERVERLESS FUNCTIONS
───────────────────────────────────────────────────────────────

  Lambda 1 ───┐
  Lambda 2 ───┤
  Lambda 3 ───┤        ┌─────────────────────┐
  Lambda 4 ───┼───────▶│    PostgreSQL       │
  Lambda 5 ───┤        │                     │
  ...         │        │  max_connections=100│
  Lambda 100 ─┘        │                     │
                       │  💥 EXHAUSTED!      │
                       └─────────────────────┘

Problem:
• Each Lambda wants its own connection
• Connections are expensive (memory, file descriptors)
• Traditional Postgres can't handle 1000s of connections
• Connection poolers (PgBouncer) help but add complexity
```

### Neon's Solution

```
NEON + SERVERLESS FUNCTIONS
───────────────────────────────────────────────────────────────

  Lambda 1 ───┐
  Lambda 2 ───┤        ┌─────────────────────┐
  Lambda 3 ───┤        │    Neon Proxy       │
  Lambda 4 ───┼───HTTP─│    (Built-in        │
  Lambda 5 ───┤        │     pooling)        │
  ...         │        └─────────────────────┘
  Lambda 1000─┘                 │
                                │ Managed
                                ▼ connections
                       ┌─────────────────────┐
                       │   Neon Compute      │
                       │   (Postgres)        │
                       └─────────────────────┘

Benefits:
• HTTP-based driver (no persistent connections)
• Built-in connection pooling
• Scales with your functions
• Pay only for what you use
```

---

## Getting Started with Neon

### 1. Create a Project

```bash
# Sign up at neon.tech (GitHub login works)
# Create a new project
# Get your connection string
```

### 2. The Connection String

```
postgresql://username:password@ep-xxx-yyy-123456.us-east-1.aws.neon.tech/neondb?sslmode=require
           │         │                    │                               │        │
           │         │                    │                               │        └─ Always required
           │         │                    │                               └─ Database name
           │         │                    └─ Endpoint (your compute)
           │         └─ Auto-generated password
           └─ Auto-generated username
```

### 3. Choose Your Driver

```
┌─────────────────────────────────────────────────────────────┐
│                    DRIVER OPTIONS                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  @neondatabase/serverless (Recommended for serverless)      │
│  ─────────────────────────────────────────────────────      │
│  • HTTP-based, works in edge/serverless                     │
│  • No connection management needed                          │
│  • Slightly higher latency per query                        │
│                                                              │
│  pg (node-postgres) + Neon Pooler                           │
│  ────────────────────────────────                           │
│  • Traditional PostgreSQL driver                            │
│  • Use with pooler endpoint (-pooler suffix)                │
│  • Better for long-running processes                        │
│                                                              │
│  Prisma, Drizzle, Kysely                                    │
│  ──────────────────────                                     │
│  • ORMs work great with Neon                                │
│  • Configure connection pooling properly                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Basic Usage with @neondatabase/serverless

```typescript
// lib/db.ts
import { neon } from '@neondatabase/serverless';

// Create a SQL function
export const sql = neon(process.env.DATABASE_URL!);

// Usage in a route handler
// app/api/users/route.ts
import { sql } from '@/lib/db';

export async function GET() {
  const users = await sql`SELECT id, name, email FROM users LIMIT 10`;
  return Response.json(users);
}

export async function POST(request: Request) {
  const { name, email } = await request.json();

  const [user] = await sql`
    INSERT INTO users (name, email)
    VALUES (${name}, ${email})
    RETURNING id, name, email
  `;

  return Response.json(user, { status: 201 });
}
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ The sql tagged template literal         │
│ automatically handles parameterization. │
│ Never concatenate user input into SQL!  │
│                                         │
│ ✓ sql`SELECT * FROM users WHERE id=${id}`│
│ ✗ sql`SELECT * FROM users WHERE id='${id}'`│
└─────────────────────────────────────────┘
```

### Using with Drizzle ORM

```typescript
// lib/db/schema.ts
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow(),
});

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content'),
  authorId: serial('author_id').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow(),
});
```

```typescript
// lib/db/index.ts
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

```typescript
// app/api/posts/route.ts
import { db } from '@/lib/db';
import { posts, users } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';

export async function GET() {
  // Type-safe queries with relations
  const postsWithAuthors = await db
    .select({
      id: posts.id,
      title: posts.title,
      author: users.name,
    })
    .from(posts)
    .leftJoin(users, eq(posts.authorId, users.id));

  return Response.json(postsWithAuthors);
}
```

### Using with Prisma

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  createdAt DateTime @default(now())
}
```

```typescript
// lib/prisma.ts
import { Pool, neonConfig } from '@neondatabase/serverless';
import { PrismaNeon } from '@prisma/adapter-neon';
import { PrismaClient } from '@prisma/client';
import ws from 'ws';

// Required for WebSocket connections
neonConfig.webSocketConstructor = ws;

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const adapter = new PrismaNeon(pool);

export const prisma = new PrismaClient({ adapter });
```

---

## Database Branching: The Game Changer

This is Neon's killer feature. Think of it like Git branches, but for your database.

```
┌─────────────────────────────────────────────────────────────┐
│                    DATABASE BRANCHING                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   main (production)                                          │
│   │                                                          │
│   │──────────────────────────────────────────▶ time          │
│   │                                                          │
│   ├─── feature/new-checkout (branch)                         │
│   │    │                                                     │
│   │    │  Full copy of production data!                      │
│   │    │  Created in ~1 second (copy-on-write)              │
│   │    │  Test migrations safely                             │
│   │    │                                                     │
│   │    └─── merge or delete                                  │
│   │                                                          │
│   ├─── preview/pr-123 (auto-created for PR)                  │
│   │                                                          │
│   └─── staging                                               │
│        │                                                     │
│        └─── Test with production-like data                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why Branching Matters

```
OLD WORKFLOW (Dangerous)
────────────────────────────────────────────────────────────

1. Write migration
2. Test on empty local database
3. Cross fingers
4. Run on production
5. 💥 Migration fails with real data
6. Scramble to fix at 2am
```

```
NEW WORKFLOW (With Neon Branching)
────────────────────────────────────────────────────────────

1. Write migration
2. Create branch from production (instant, free)
3. Run migration on branch (with real data!)
4. Test thoroughly
5. If it works → apply to production
6. If it fails → delete branch, fix, try again
7. Sleep peacefully
```

### Creating Branches

```bash
# Using Neon CLI
neon branches create --name feature/checkout

# Using Neon API (for CI/CD)
curl -X POST 'https://console.neon.tech/api/v2/projects/{project_id}/branches' \
  -H 'Authorization: Bearer $NEON_API_KEY' \
  -d '{"branch": {"name": "feature/checkout", "parent_id": "br-xxx"}}'
```

### Vercel + Neon Branching Integration

```
┌─────────────────────────────────────────────────────────────┐
│              AUTOMATIC PREVIEW DATABASES                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Developer opens PR                                      │
│                                                              │
│   2. Vercel Integration triggers:                            │
│      • Creates Neon branch automatically                     │
│      • Names it: preview/pr-123                              │
│      • Sets DATABASE_URL for preview deployment              │
│                                                              │
│   3. Preview deployment uses isolated database               │
│      • Test migrations without affecting production          │
│      • Test with real data structure                         │
│                                                              │
│   4. PR merged or closed                                     │
│      • Branch automatically deleted                          │
│      • No orphaned resources                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Connection Pooling Deep Dive

Understanding connection modes is crucial:

```
┌─────────────────────────────────────────────────────────────┐
│                CONNECTION ENDPOINTS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DIRECT CONNECTION                                           │
│  ─────────────────                                           │
│  postgresql://user:pass@ep-xxx.neon.tech/db                 │
│                                                              │
│  • Direct to compute                                         │
│  • Best for: migrations, schema changes                      │
│  • ⚠️ Not recommended for serverless                         │
│                                                              │
│  POOLED CONNECTION                                           │
│  ─────────────────                                           │
│  postgresql://user:pass@ep-xxx-pooler.neon.tech/db          │
│                        ^^^^^^^^                              │
│                        Add -pooler                           │
│                                                              │
│  • Goes through PgBouncer                                    │
│  • Best for: application queries                             │
│  • ✓ Recommended for serverless                              │
│                                                              │
│  SERVERLESS DRIVER (HTTP)                                    │
│  ─────────────────────────                                   │
│  @neondatabase/serverless                                    │
│                                                              │
│  • HTTP-based, no persistent connections                     │
│  • Best for: Edge functions, Vercel Edge Runtime             │
│  • ✓ Ideal for serverless                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// Recommended setup for Next.js
// lib/db.ts

import { neon, neonConfig } from '@neondatabase/serverless';

// For Edge Runtime
export const sql = neon(process.env.DATABASE_URL!);

// For Node.js Runtime with pooling
import { Pool } from '@neondatabase/serverless';

neonConfig.fetchConnectionCache = true; // Reuse connections

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10, // Maximum connections in pool
});
```

---

## Scale to Zero Explained

```
┌─────────────────────────────────────────────────────────────┐
│                    SCALE TO ZERO                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Traffic Pattern:                                           │
│                                                              │
│     │ ┌───┐                    ┌──┐     ┌─┐                 │
│   Q │ │   │                    │  │     │ │                 │
│   u │ │   │                    │  │     │ │                 │
│   e │ │   └──┐        ┌───┐   │  │     │ │                 │
│   r │ │      └────────┘   └───┘  └─────┘ └────────────     │
│   i │ │                                                     │
│   e │─┼─────────────────────────────────────────────────▶   │
│   s │                        Time                           │
│                                                              │
│   Compute Usage:                                             │
│                                                              │
│     │ ┌───┐                    ┌──┐     ┌─┐                 │
│   C │ │   │                    │  │     │ │                 │
│   P │ │   │                    │  │     │ │                 │
│   U │ │   └──┐        ┌───┐   │  │     │ │                 │
│     │ │      └────────┘   └───┘  └─────┘ └────────────     │
│     │─┼─────────────────────────────────────────────────▶   │
│     │ ─────── Zero when inactive ─────────                  │
│                                                              │
│   What you pay for: Only the colored areas                  │
│   Auto-suspend after: 5 minutes of inactivity (configurable)│
│   Wake-up time: ~500ms (first query)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ For production with consistent traffic, │
│ disable auto-suspend to avoid cold      │
│ starts:                                 │
│                                         │
│ Neon Console → Compute → Settings →     │
│ Auto-suspend delay → Disabled           │
│                                         │
│ For dev/preview branches, keep it on    │
│ to save costs.                          │
└─────────────────────────────────────────┘
```

---

## Autoscaling

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTOSCALING                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Configure min/max compute units:                           │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  MIN: 0.25 CU                   MAX: 4 CU           │   │
│   │       │                              │              │   │
│   │       │◀── Scales automatically ───▶│              │   │
│   │       │                              │              │   │
│   │  Low traffic               High traffic             │   │
│   │  = low cost                = high performance       │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   1 CU (Compute Unit) ≈ 1 vCPU, 4GB RAM                     │
│                                                              │
│   Example configurations:                                    │
│   • Development:  0.25 - 0.25 CU (always minimal)           │
│   • Staging:      0.25 - 1 CU (scales when needed)          │
│   • Production:   1 - 4 CU (never below 1 CU)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Pricing Breakdown

```
┌─────────────────────────────────────────────────────────────┐
│                    NEON PRICING (2025)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  FREE TIER                                                  │
│  ─────────                                                  │
│  • 0.5 GB storage                                           │
│  • 1 project                                                │
│  • 10 branches                                              │
│  • Primary compute: 0.25 CU                                 │
│  • 191.9 compute hours/month (~24/7 at 0.25 CU)            │
│  • 👍 Perfect for learning and small projects               │
│                                                              │
│  LAUNCH ($19/month)                                         │
│  ─────────────────                                          │
│  • 10 GB storage                                            │
│  • Unlimited projects                                       │
│  • Unlimited branches                                       │
│  • Up to 4 CU                                               │
│  • 300 compute hours included                               │
│  • Point-in-time recovery: 7 days                          │
│                                                              │
│  SCALE ($69/month)                                          │
│  ─────────────────                                          │
│  • 50 GB storage included                                   │
│  • Up to 8 CU                                               │
│  • 750 compute hours included                               │
│  • Read replicas                                            │
│  • Point-in-time recovery: 14 days                         │
│                                                              │
│  ENTERPRISE (Custom)                                        │
│  ──────────────────                                         │
│  • SLA, dedicated support                                   │
│  • Custom limits                                            │
│  • SOC 2 compliance                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Exercise: Set Up Neon with Next.js

```
┌─────────────────────────────────────────┐
│ 🎯 EXERCISE                             │
│                                         │
│ 1. Sign up at neon.tech                 │
│                                         │
│ 2. Create a new project                 │
│                                         │
│ 3. Copy the connection string           │
│                                         │
│ 4. Add to your .env.local:              │
│    DATABASE_URL=postgresql://...        │
│                                         │
│ 5. Install the driver:                  │
│    npm install @neondatabase/serverless │
│                                         │
│ 6. Create lib/db.ts:                    │
│    import { neon } from '...'           │
│    export const sql = neon(...)         │
│                                         │
│ 7. Create a table in Neon console:      │
│    CREATE TABLE posts (                 │
│      id SERIAL PRIMARY KEY,             │
│      title TEXT NOT NULL,               │
│      created_at TIMESTAMP DEFAULT NOW() │
│    );                                   │
│                                         │
│ 8. Query from an API route:             │
│    const posts = await sql`             │
│      SELECT * FROM posts                │
│    `;                                   │
│                                         │
│ BONUS: Create a branch, add test data,  │
│ then delete the branch.                 │
└─────────────────────────────────────────┘
```

---

## Common Mistakes to Avoid

```
┌─────────────────────────────────────────┐
│ ⚠️  WARNING                             │
│                                         │
│ 1. Using direct connection in serverless│
│    → Use pooled endpoint or HTTP driver │
│                                         │
│ 2. Not handling cold starts             │
│    → Keep compute warm in production    │
│    → Use connection retries             │
│                                         │
│ 3. Running migrations with pooler       │
│    → Use direct connection for DDL      │
│                                         │
│ 4. Ignoring connection limits           │
│    → Set appropriate pool sizes         │
│    → Monitor connection usage           │
│                                         │
│ 5. Not using branches for testing       │
│    → Always test migrations on branches │
└─────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Neon is Postgres** — Everything you know about Postgres works
2. **Branching is revolutionary** — Test with production data safely
3. **Scale to zero saves money** — Pay only for active compute
4. **Use the right driver** — HTTP for serverless, pooled for apps
5. **Connection management matters** — Pooling is your friend

---

## What's Next?

With your database sorted, you'll often need fast caching and background jobs. In Chapter 4, we'll explore **Upstash** — serverless Redis and Kafka that complement Neon perfectly.

---

[← Previous: Vercel](02-vercel.md) | [Back to Contents](../README.md) | [Next: Upstash →](04-upstash.md)
