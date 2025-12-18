# Chapter 5: From Express to Fastify

> "The best code is code you don't have to write."

---

## The Express Era

Express.js has been the **default Node.js web framework** for over a decade. It's minimal, flexible, and has a massive ecosystem. So why are developers moving to Fastify?

```
EXPRESS TIMELINE
───────────────────────────────────────────────────────────────

2010 ─────────────────────────────────────────────────────────▶

     │         Express born
     │         (TJ Holowaychuk)
     ▼
   2010        2014         2017         2020         2024
     │           │            │            │            │
     │           │            │            │            │
     │      Express 4    Koa rises    Fastify     Fastify
     │      (still     (async/await)  gains       dominant
     │       used                     traction     in new
     │       today!)                              projects

```

---

## Why Express is Showing Its Age

### 1. Performance

```
BENCHMARK: Requests Per Second (Higher = Better)
──────────────────────────────────────────────────────────────

Fastify     ████████████████████████████████████████  78,956
Koa         ██████████████████████████████            54,846
Express     ████████████████████                      38,510
Hapi        ███████████████                           29,998

Source: fastify.dev/benchmarks (2024)
```

**Why the difference?**

```javascript
// EXPRESS: Builds middleware array, walks through it for EVERY request
app.use(cors());
app.use(bodyParser.json());
app.use(authMiddleware);
app.use(loggingMiddleware);
// ... every request walks this chain

// FASTIFY: Compiles routes into optimized radix tree at startup
// Uses find-my-way router (extremely fast route matching)
// Schema-based validation compiled to optimized functions
```

### 2. TypeScript Support

```typescript
// EXPRESS: Types are an afterthought
import express, { Request, Response } from 'express';

// No type safety on req.body, req.params, req.query
app.post('/users', (req: Request, res: Response) => {
  const { name, email } = req.body; // any!
  // You have to trust the client sent correct data
  // Or add manual validation + type assertions
});
```

```typescript
// FASTIFY: Types are first-class
import Fastify from 'fastify';

const app = Fastify();

// Define schema + types together
interface CreateUserBody {
  name: string;
  email: string;
}

app.post<{ Body: CreateUserBody }>(
  '/users',
  {
    schema: {
      body: {
        type: 'object',
        required: ['name', 'email'],
        properties: {
          name: { type: 'string' },
          email: { type: 'string', format: 'email' },
        },
      },
    },
  },
  async (request, reply) => {
    // request.body is typed as CreateUserBody
    // AND validated at runtime!
    const { name, email } = request.body;
  }
);
```

### 3. Built-in Validation

```javascript
// EXPRESS: You need external libraries
import express from 'express';
import { body, validationResult } from 'express-validator';
import Joi from 'joi';
// or zod, yup, etc.

app.post(
  '/users',
  body('email').isEmail(),
  body('name').notEmpty(),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // ... proceed
  }
);
```

```javascript
// FASTIFY: Validation is built-in (JSON Schema)
app.post(
  '/users',
  {
    schema: {
      body: {
        type: 'object',
        required: ['name', 'email'],
        properties: {
          name: { type: 'string', minLength: 1 },
          email: { type: 'string', format: 'email' },
        },
      },
      response: {
        201: {
          type: 'object',
          properties: {
            id: { type: 'number' },
            name: { type: 'string' },
          },
        },
      },
    },
  },
  async (request, reply) => {
    // Already validated! Fastify returns 400 automatically if invalid
    const { name, email } = request.body;
    return reply.code(201).send({ id: 1, name });
  }
);
```

### 4. Async/Await Support

```javascript
// EXPRESS: Error handling is painful
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.findUser(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'Not found' });
    }
    res.json(user);
  } catch (error) {
    next(error); // Easy to forget!
  }
});

// Need error-handling middleware
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Internal error' });
});
```

```javascript
// FASTIFY: Just return or throw
app.get('/users/:id', async (request, reply) => {
  const user = await db.findUser(request.params.id);

  if (!user) {
    reply.code(404);
    return { error: 'Not found' };
  }

  return user; // Automatic JSON serialization
  // Errors are caught automatically
});

// Or throw errors directly
app.get('/users/:id', async (request, reply) => {
  const user = await db.findUser(request.params.id);

  if (!user) {
    throw app.httpErrors.notFound('User not found');
  }

  return user;
});
```

---

## Fastify Core Concepts

### 1. The Plugin System

Fastify's encapsulation model is its superpower:

```
┌─────────────────────────────────────────────────────────────┐
│                    FASTIFY PLUGIN SYSTEM                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  ROOT (app)                                         │   │
│   │                                                     │   │
│   │  ┌─────────────────────────────────────────────┐   │   │
│   │  │  Plugin: /api                               │   │   │
│   │  │  (has its own decorators, hooks)            │   │   │
│   │  │                                             │   │   │
│   │  │  ┌───────────────────────────────────────┐ │   │   │
│   │  │  │  Plugin: /api/users                   │ │   │   │
│   │  │  │  (inherits from parent, can override) │ │   │   │
│   │  │  └───────────────────────────────────────┘ │   │   │
│   │  │                                             │   │   │
│   │  │  ┌───────────────────────────────────────┐ │   │   │
│   │  │  │  Plugin: /api/posts                   │ │   │   │
│   │  │  │  (isolated from /api/users)           │ │   │   │
│   │  │  └───────────────────────────────────────┘ │   │   │
│   │  │                                             │   │   │
│   │  └─────────────────────────────────────────────┘   │   │
│   │                                                     │   │
│   │  ┌─────────────────────────────────────────────┐   │   │
│   │  │  Plugin: /admin                             │   │   │
│   │  │  (completely isolated from /api)            │   │   │
│   │  └─────────────────────────────────────────────┘   │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   Each plugin has its own context:                          │
│   • Decorators don't leak to siblings                       │
│   • Hooks are scoped                                        │
│   • Perfect for microservices & modular apps                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// plugins/database.ts
import fp from 'fastify-plugin';
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';

export default fp(async (fastify) => {
  const sql = neon(process.env.DATABASE_URL!);
  const db = drizzle(sql);

  // Decorate fastify instance
  fastify.decorate('db', db);
});

// Now app.db is available everywhere
```

```typescript
// plugins/auth.ts
import fp from 'fastify-plugin';

export default fp(async (fastify) => {
  fastify.decorateRequest('user', null);

  fastify.addHook('preHandler', async (request) => {
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (token) {
      try {
        request.user = await verifyToken(token);
      } catch {
        // Invalid token, user remains null
      }
    }
  });
});
```

### 2. Hooks (Lifecycle)

```
┌─────────────────────────────────────────────────────────────┐
│                    REQUEST LIFECYCLE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Incoming Request                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ onRequest   │  ← Logging, rate limiting                 │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ preParsing  │  ← Modify raw payload                     │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ preValidation│ ← Before schema validation               │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ preHandler  │  ← Auth checks, load resources            │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │   Handler   │  ← Your route handler                     │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ preSerialization│ ← Modify response before JSON        │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ onSend      │  ← Modify serialized payload              │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   ┌─────────────┐                                           │
│   │ onResponse  │  ← Logging, metrics                       │
│   └─────────────┘                                           │
│        │                                                     │
│        ▼                                                     │
│   Response Sent                                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3. Schema Validation & Serialization

```typescript
const userSchema = {
  type: 'object',
  required: ['name', 'email'],
  properties: {
    name: { type: 'string', minLength: 1, maxLength: 100 },
    email: { type: 'string', format: 'email' },
    age: { type: 'integer', minimum: 0, maximum: 150 },
    role: { type: 'string', enum: ['user', 'admin'] },
  },
} as const;

app.post('/users', {
  schema: {
    body: userSchema,
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' },
        },
      },
      400: {
        type: 'object',
        properties: {
          error: { type: 'string' },
        },
      },
    },
  },
  handler: async (request, reply) => {
    const user = await createUser(request.body);
    return reply.code(201).send(user);
  },
});
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Response schemas aren't just for docs—  │
│ they ALSO improve performance!          │
│                                         │
│ Fastify compiles response schemas into  │
│ optimized serializers. This can be 2-3x │
│ faster than JSON.stringify().           │
│                                         │
│ Also: they prevent accidentally leaking │
│ sensitive data (passwords, tokens).     │
└─────────────────────────────────────────┘
```

---

## Migration Guide: Express to Fastify

### Before (Express)

```typescript
// app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { body, validationResult } from 'express-validator';

const app = express();

app.use(cors());
app.use(helmet());
app.use(express.json());

// Middleware
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  req.user = verifyToken(token);
  next();
};

// Routes
app.get('/users', authMiddleware, async (req, res, next) => {
  try {
    const users = await db.query('SELECT * FROM users');
    res.json(users);
  } catch (error) {
    next(error);
  }
});

app.post(
  '/users',
  authMiddleware,
  body('email').isEmail(),
  body('name').notEmpty(),
  async (req, res, next) => {
    try {
      const errors = validationResult(req);
      if (!errors.isEmpty()) {
        return res.status(400).json({ errors: errors.array() });
      }

      const user = await db.query(
        'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
        [req.body.name, req.body.email]
      );

      res.status(201).json(user);
    } catch (error) {
      next(error);
    }
  }
);

// Error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Internal error' });
});

app.listen(3000);
```

### After (Fastify)

```typescript
// app.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import helmet from '@fastify/helmet';

const app = Fastify({
  logger: true, // Built-in structured logging!
});

// Plugins (cleaner than app.use())
await app.register(cors);
await app.register(helmet);

// Custom plugin for auth
app.register(async (fastify) => {
  fastify.decorateRequest('user', null);

  fastify.addHook('preHandler', async (request, reply) => {
    const token = request.headers.authorization;
    if (!token) {
      return reply.code(401).send({ error: 'Unauthorized' });
    }
    request.user = await verifyToken(token);
  });

  // Routes that need auth
  fastify.get('/users', async (request) => {
    return db.query('SELECT * FROM users');
  });

  fastify.post<{
    Body: { name: string; email: string };
  }>(
    '/users',
    {
      schema: {
        body: {
          type: 'object',
          required: ['name', 'email'],
          properties: {
            name: { type: 'string', minLength: 1 },
            email: { type: 'string', format: 'email' },
          },
        },
        response: {
          201: {
            type: 'object',
            properties: {
              id: { type: 'number' },
              name: { type: 'string' },
              email: { type: 'string' },
            },
          },
        },
      },
    },
    async (request, reply) => {
      const user = await db.query(
        'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
        [request.body.name, request.body.email]
      );

      return reply.code(201).send(user);
    }
  );
});

// Errors are handled automatically, but you can customize:
app.setErrorHandler((error, request, reply) => {
  request.log.error(error);
  reply.code(500).send({ error: 'Internal error' });
});

await app.listen({ port: 3000 });
```

---

## Bonus: Hono — The Edge-First Framework

While Fastify dominates Node.js, **Hono** is emerging for edge/serverless:

```typescript
import { Hono } from 'hono';

const app = new Hono();

app.get('/', (c) => c.text('Hello Hono!'));

app.get('/api/users/:id', async (c) => {
  const id = c.req.param('id');
  const user = await getUser(id);
  return c.json(user);
});

// Works on:
// • Cloudflare Workers
// • Vercel Edge
// • Deno Deploy
// • Bun
// • Node.js

export default app;
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Fastify vs Hono:                        │
│                                         │
│ Fastify:                                │
│ • Full Node.js APIs available           │
│ • Rich plugin ecosystem                 │
│ • Best for traditional backends         │
│                                         │
│ Hono:                                   │
│ • Runs everywhere (edge, Bun, etc.)     │
│ • Tiny bundle size (~14kb)              │
│ • Best for edge-first apps              │
│                                         │
│ Both are excellent choices in 2026.     │
└─────────────────────────────────────────┘
```

---

## Exercise: Build a Fastify API

```
┌─────────────────────────────────────────┐
│ 🎯 EXERCISE                             │
│                                         │
│ Create a simple Fastify API:            │
│                                         │
│ 1. Initialize project:                  │
│    mkdir fastify-api && cd fastify-api  │
│    npm init -y                          │
│    npm install fastify                  │
│    npm install -D typescript @types/node│
│                                         │
│ 2. Create src/index.ts:                 │
│    • Initialize Fastify with logger     │
│    • Add GET /health → { status: 'ok' } │
│    • Add POST /echo → returns body      │
│    • Add schema validation to /echo     │
│                                         │
│ 3. Add a preHandler hook:               │
│    • Log request method and URL         │
│                                         │
│ 4. Run and test:                        │
│    npx tsx src/index.ts                 │
│    curl localhost:3000/health           │
│    curl -X POST localhost:3000/echo \   │
│      -H "Content-Type: application/json"│
│      -d '{"message": "hello"}'          │
│                                         │
│ BONUS: Add rate limiting with           │
│ @fastify/rate-limit plugin              │
└─────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Fastify is 2x faster than Express** — And that matters at scale
2. **Built-in validation saves time** — No more validator library fatigue
3. **TypeScript is first-class** — Real type safety, not just type hints
4. **Plugin system enables clean architecture** — Encapsulation by default
5. **Express isn't dead** — But new projects should consider Fastify

---

## When to Still Use Express

- Massive existing codebase (migration cost too high)
- Team deeply familiar with Express patterns
- Specific middleware only available for Express
- Quick prototypes (Express's simplicity still shines)

---

## What's Next?

We've covered backend frameworks, but most modern apps are frontend-heavy. In Chapter 6, we'll master **Next.js** — the React framework that brings server and client together.

---

[← Previous: Upstash](04-upstash.md) | [Back to Contents](../README.md) | [Next: Next.js →](06-nextjs.md)
