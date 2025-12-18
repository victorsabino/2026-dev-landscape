# Chapter 8: Best Practices & Workflows

> "Excellence is not an act, but a habit." — Aristotle

---

## The Complete Modern Stack

Let's bring together everything we've learned:

```
┌─────────────────────────────────────────────────────────────┐
│                    THE 2026 STACK                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   FRONTEND                                                   │
│   ────────                                                   │
│   • Next.js 15 (App Router)                                 │
│   • React 19 (Server Components)                            │
│   • TypeScript (strict mode)                                │
│   • TailwindCSS / CSS Modules                               │
│                                                              │
│   BACKEND                                                    │
│   ───────                                                    │
│   • Next.js API Routes (simple APIs)                        │
│   • Fastify (complex backends)                              │
│   • tRPC (type-safe APIs)                                   │
│                                                              │
│   DATABASE                                                   │
│   ────────                                                   │
│   • Neon (serverless Postgres)                              │
│   • Drizzle ORM (type-safe queries)                         │
│   • Prisma (for complex schemas)                            │
│                                                              │
│   CACHE / QUEUE                                              │
│   ────────────                                               │
│   • Upstash Redis (caching, rate limiting)                  │
│   • Upstash QStash (background jobs)                        │
│                                                              │
│   DEPLOYMENT                                                 │
│   ──────────                                                 │
│   • Vercel (frontend + serverless)                          │
│   • Railway / Render (containers)                           │
│                                                              │
│   TOOLS                                                      │
│   ─────                                                      │
│   • Claude Code (AI assistance)                             │
│   • GitHub Actions (CI/CD)                                  │
│   • Turborepo (monorepos)                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Development Workflow

### 1. Project Setup

```bash
# Create new project
npx create-next-app@latest my-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"

cd my-app

# Add essential dependencies
npm install @neondatabase/serverless    # Database
npm install @upstash/redis             # Cache
npm install @upstash/ratelimit         # Rate limiting
npm install drizzle-orm                # ORM
npm install zod                        # Validation

# Dev dependencies
npm install -D drizzle-kit             # Migrations
npm install -D prettier                # Formatting
npm install -D @types/node
```

### 2. Environment Setup

```bash
# .env.local (never commit this!)
DATABASE_URL=postgresql://...
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...

# .env.example (commit this as template)
DATABASE_URL=
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
```

### 3. Project Structure

```
my-app/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/            # Route group (no URL impact)
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (dashboard)/       # Another route group
│   │   │   └── dashboard/
│   │   ├── api/               # API routes
│   │   ├── layout.tsx
│   │   └── page.tsx
│   │
│   ├── components/            # React components
│   │   ├── ui/               # Generic UI components
│   │   └── features/         # Feature-specific components
│   │
│   ├── lib/                  # Utilities and configurations
│   │   ├── db/              # Database client & schema
│   │   ├── redis.ts         # Redis client
│   │   ├── auth.ts          # Auth utilities
│   │   └── utils.ts         # General utilities
│   │
│   ├── hooks/               # Custom React hooks
│   ├── types/               # TypeScript types
│   └── styles/              # Global styles
│
├── public/                  # Static assets
├── drizzle/                 # Database migrations
├── tests/                   # Test files
├── .env.local              # Local environment (gitignored)
├── .env.example            # Environment template
├── drizzle.config.ts       # Drizzle configuration
└── next.config.js          # Next.js configuration
```

---

## TypeScript Best Practices

### Strict Mode Always

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### Define Clear Types

```typescript
// types/index.ts

// Use interfaces for objects that can be extended
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}

// Use type aliases for unions, intersections, utilities
type UserRole = 'admin' | 'user' | 'guest';

type CreateUserInput = Pick<User, 'email' | 'name'>;
type UpdateUserInput = Partial<CreateUserInput>;

// API response types
interface ApiResponse<T> {
  data: T;
  error?: never;
}

interface ApiError {
  data?: never;
  error: {
    message: string;
    code: string;
  };
}

type ApiResult<T> = ApiResponse<T> | ApiError;
```

### Validate at Boundaries

```typescript
// lib/validations.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;

// app/api/users/route.ts
import { createUserSchema } from '@/lib/validations';

export async function POST(request: Request) {
  const body = await request.json();

  // Validate at the boundary
  const result = createUserSchema.safeParse(body);

  if (!result.success) {
    return Response.json(
      { error: result.error.flatten() },
      { status: 400 }
    );
  }

  // result.data is now typed and validated
  const user = await createUser(result.data);
  return Response.json(user, { status: 201 });
}
```

---

## Error Handling

### Consistent Error Types

```typescript
// lib/errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404);
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor() {
    super('Unauthorized', 'UNAUTHORIZED', 401);
  }
}
```

### Error Handling in API Routes

```typescript
// lib/api-handler.ts
import { AppError } from './errors';

type Handler = (request: Request) => Promise<Response>;

export function withErrorHandling(handler: Handler): Handler {
  return async (request: Request) => {
    try {
      return await handler(request);
    } catch (error) {
      console.error('API Error:', error);

      if (error instanceof AppError) {
        return Response.json(
          { error: { message: error.message, code: error.code } },
          { status: error.statusCode }
        );
      }

      return Response.json(
        { error: { message: 'Internal server error', code: 'INTERNAL_ERROR' } },
        { status: 500 }
      );
    }
  };
}

// Usage
// app/api/users/[id]/route.ts
import { withErrorHandling } from '@/lib/api-handler';
import { NotFoundError } from '@/lib/errors';

export const GET = withErrorHandling(async (request, { params }) => {
  const user = await getUser(params.id);

  if (!user) {
    throw new NotFoundError('User');
  }

  return Response.json(user);
});
```

---

## Database Patterns

### Repository Pattern

```typescript
// lib/db/repositories/user-repository.ts
import { db } from '../index';
import { users, type User, type NewUser } from '../schema';
import { eq } from 'drizzle-orm';

export const userRepository = {
  async findById(id: number): Promise<User | null> {
    const [user] = await db
      .select()
      .from(users)
      .where(eq(users.id, id))
      .limit(1);
    return user ?? null;
  },

  async findByEmail(email: string): Promise<User | null> {
    const [user] = await db
      .select()
      .from(users)
      .where(eq(users.email, email))
      .limit(1);
    return user ?? null;
  },

  async create(data: NewUser): Promise<User> {
    const [user] = await db.insert(users).values(data).returning();
    return user;
  },

  async update(id: number, data: Partial<NewUser>): Promise<User | null> {
    const [user] = await db
      .update(users)
      .set(data)
      .where(eq(users.id, id))
      .returning();
    return user ?? null;
  },

  async delete(id: number): Promise<boolean> {
    const result = await db.delete(users).where(eq(users.id, id));
    return result.rowCount > 0;
  },
};
```

### Service Layer

```typescript
// lib/services/user-service.ts
import { userRepository } from '../db/repositories/user-repository';
import { NotFoundError, ValidationError } from '../errors';
import { hashPassword, verifyPassword } from '../auth';
import type { CreateUserInput } from '../validations';

export const userService = {
  async createUser(input: CreateUserInput) {
    // Check if email exists
    const existing = await userRepository.findByEmail(input.email);
    if (existing) {
      throw new ValidationError('Email already registered');
    }

    // Hash password
    const hashedPassword = await hashPassword(input.password);

    // Create user
    return userRepository.create({
      email: input.email,
      name: input.name,
      passwordHash: hashedPassword,
    });
  },

  async getUser(id: number) {
    const user = await userRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User');
    }
    return user;
  },

  async authenticateUser(email: string, password: string) {
    const user = await userRepository.findByEmail(email);
    if (!user) {
      throw new ValidationError('Invalid credentials');
    }

    const valid = await verifyPassword(password, user.passwordHash);
    if (!valid) {
      throw new ValidationError('Invalid credentials');
    }

    return user;
  },
};
```

---

## Testing Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    TESTING PYRAMID                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                        /\                                    │
│                       /  \                                   │
│                      / E2E \        Few, slow, expensive     │
│                     /──────\                                 │
│                    /        \                                │
│                   /Integration\    Some, medium speed        │
│                  /────────────\                              │
│                 /              \                             │
│                /     Unit       \   Many, fast, cheap        │
│               /──────────────────\                           │
│                                                              │
│   Unit Tests: Individual functions, utilities, hooks         │
│   Integration Tests: API routes, database operations         │
│   E2E Tests: Critical user flows (checkout, auth)           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Unit Tests with Vitest

```typescript
// lib/utils.test.ts
import { describe, it, expect } from 'vitest';
import { formatCurrency, slugify, truncate } from './utils';

describe('formatCurrency', () => {
  it('formats USD correctly', () => {
    expect(formatCurrency(1234.56, 'USD')).toBe('$1,234.56');
  });

  it('handles zero', () => {
    expect(formatCurrency(0, 'USD')).toBe('$0.00');
  });

  it('handles negative numbers', () => {
    expect(formatCurrency(-50, 'USD')).toBe('-$50.00');
  });
});

describe('slugify', () => {
  it('converts spaces to hyphens', () => {
    expect(slugify('Hello World')).toBe('hello-world');
  });

  it('removes special characters', () => {
    expect(slugify('Hello! World?')).toBe('hello-world');
  });

  it('handles multiple spaces', () => {
    expect(slugify('Hello   World')).toBe('hello-world');
  });
});
```

### Integration Tests

```typescript
// app/api/users/route.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { createTestDatabase, cleanupTestDatabase } from '@/tests/helpers';

describe('POST /api/users', () => {
  beforeEach(async () => {
    await createTestDatabase();
  });

  afterEach(async () => {
    await cleanupTestDatabase();
  });

  it('creates a user with valid data', async () => {
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'test@example.com',
        name: 'Test User',
        password: 'securepass123',
      }),
    });

    expect(response.status).toBe(201);
    const data = await response.json();
    expect(data.email).toBe('test@example.com');
    expect(data.password).toBeUndefined(); // Should not expose password
  });

  it('returns 400 for invalid email', async () => {
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'invalid-email',
        name: 'Test User',
        password: 'securepass123',
      }),
    });

    expect(response.status).toBe(400);
  });
});
```

---

## Git Workflow

### Branch Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    BRANCH STRATEGY                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   main                                                       │
│   │                                                          │
│   ├─── Always deployable                                    │
│   │    Protected branch (require PR + reviews)              │
│   │                                                          │
│   ├─── feature/user-auth                                    │
│   │    └── New feature development                          │
│   │                                                          │
│   ├─── fix/checkout-bug                                     │
│   │    └── Bug fixes                                        │
│   │                                                          │
│   └─── chore/update-deps                                    │
│        └── Maintenance tasks                                │
│                                                              │
│   Branch naming:                                            │
│   • feature/ — New features                                 │
│   • fix/ — Bug fixes                                        │
│   • chore/ — Maintenance                                    │
│   • docs/ — Documentation                                   │
│   • refactor/ — Code improvements                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Commit Messages

```
┌─────────────────────────────────────────────────────────────┐
│                    COMMIT MESSAGE FORMAT                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   type(scope): description                                   │
│                                                              │
│   Types:                                                     │
│   • feat: New feature                                       │
│   • fix: Bug fix                                            │
│   • docs: Documentation                                      │
│   • style: Formatting (no code change)                      │
│   • refactor: Code restructuring                            │
│   • test: Adding tests                                       │
│   • chore: Maintenance                                       │
│                                                              │
│   Examples:                                                  │
│   feat(auth): add password reset flow                       │
│   fix(checkout): resolve race condition in payment          │
│   docs(api): update endpoint documentation                  │
│   refactor(users): extract validation to separate module    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

---

## Security Checklist

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY CHECKLIST                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ☐ Environment variables for secrets (never hardcode)       │
│  ☐ Input validation on all endpoints (Zod)                  │
│  ☐ Parameterized queries (never string concatenation)       │
│  ☐ Rate limiting on public endpoints                        │
│  ☐ CORS configured correctly                                │
│  ☐ Authentication on protected routes                       │
│  ☐ Authorization checks (user can only access their data)   │
│  ☐ HTTPS only (Vercel handles this)                        │
│  ☐ Secure cookies (HttpOnly, Secure, SameSite)             │
│  ☐ No sensitive data in URLs or logs                        │
│  ☐ Dependency audit (npm audit)                             │
│  ☐ CSP headers configured                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Performance Checklist

```
┌─────────────────────────────────────────────────────────────┐
│                    PERFORMANCE CHECKLIST                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  FRONTEND                                                    │
│  ────────                                                    │
│  ☐ Server Components for static content                     │
│  ☐ Dynamic imports for large components                     │
│  ☐ Image optimization (next/image)                          │
│  ☐ Font optimization (next/font)                            │
│  ☐ Minimize client-side JavaScript                          │
│                                                              │
│  BACKEND                                                     │
│  ───────                                                     │
│  ☐ Database indexes on frequently queried columns           │
│  ☐ Connection pooling (Neon pooler)                         │
│  ☐ Cache expensive queries (Upstash Redis)                  │
│  ☐ Pagination for list endpoints                            │
│  ☐ Select only needed columns                               │
│                                                              │
│  INFRASTRUCTURE                                              │
│  ──────────────                                              │
│  ☐ Edge functions for latency-sensitive routes              │
│  ☐ CDN for static assets                                    │
│  ☐ Database region close to compute                         │
│  ☐ Monitoring and alerting                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## The Developer Mindset

### Continuous Learning

```
┌─────────────────────────────────────────┐
│ 💡 The tech landscape changes fast.     │
│                                         │
│ Dedicate time each week to:             │
│ • Read release notes of tools you use   │
│ • Follow key people on Twitter/X        │
│ • Try new tools in side projects        │
│ • Contribute to open source             │
│                                         │
│ Stay curious, but don't chase every     │
│ new thing. Depth beats breadth.         │
└─────────────────────────────────────────┘
```

### Ship Early, Iterate Often

```
┌─────────────────────────────────────────┐
│ "If you're not embarrassed by the      │
│  first version of your product,        │
│  you've launched too late."            │
│                                         │
│                    — Reid Hoffman        │
│                                         │
│ • Start with MVP                        │
│ • Get user feedback                     │
│ • Iterate based on data                 │
│ • Perfect later (or never)              │
└─────────────────────────────────────────┘
```

### Write Code for Humans

```
┌─────────────────────────────────────────┐
│ "Any fool can write code that a        │
│  computer can understand. Good          │
│  programmers write code that humans     │
│  can understand."                       │
│                                         │
│                    — Martin Fowler       │
│                                         │
│ • Clear names > clever code             │
│ • Simple > complex                      │
│ • Explicit > implicit                   │
│ • Comments explain "why", not "what"    │
└─────────────────────────────────────────┘
```

---

## Final Exercise: Build a Complete App

```
┌─────────────────────────────────────────┐
│ 🎯 CAPSTONE PROJECT                     │
│                                         │
│ Build a "Link Shortener" with:          │
│                                         │
│ TECH STACK                              │
│ ──────────                              │
│ • Next.js 15 (App Router)               │
│ • Neon (database)                       │
│ • Upstash Redis (caching + rate limit)  │
│ • Vercel (deployment)                   │
│                                         │
│ FEATURES                                │
│ ────────                                │
│ • Create shortened URLs                 │
│ • Redirect with analytics tracking      │
│ • Dashboard showing click stats         │
│ • Rate limiting (10 creates/minute)     │
│ • User authentication                   │
│                                         │
│ REQUIREMENTS                            │
│ ────────────                            │
│ • TypeScript throughout                 │
│ • Input validation with Zod             │
│ • Error handling                        │
│ • Unit tests for utilities              │
│ • Integration tests for API             │
│ • CI pipeline with GitHub Actions       │
│ • Preview deployments on PRs            │
│                                         │
│ Use Claude Code to help you build it!   │
│                                         │
└─────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Use the modern stack** — Vercel + Neon + Upstash is production-ready
2. **TypeScript strict mode** — Catch bugs before they ship
3. **Validate at boundaries** — Trust nothing from outside
4. **Test the important stuff** — Not everything, but the critical paths
5. **Ship and iterate** — Done is better than perfect
6. **Use AI wisely** — It's a tool, not a replacement for understanding

---

## Conclusion

You've completed The 2026 Developer's Mentor. You now understand:

- **Modern infrastructure** (Vercel, Neon, Upstash)
- **Modern frameworks** (Next.js, Fastify)
- **AI assistance** (Claude Code)
- **Best practices** (TypeScript, testing, security)

The tools will continue to evolve, but the principles remain:

- Build for users, not for technology
- Ship early, learn fast
- Write code that others can understand
- Never stop learning

**Now go build something amazing.**

---

[← Previous: Claude Code](07-claude-code.md) | [Back to Contents](../README.md)

---

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║          Thank you for reading!                             ║
║                                                              ║
║          The best way to learn is to build.                 ║
║          Start your next project today.                     ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```
