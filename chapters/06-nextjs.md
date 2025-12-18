# Chapter 6: Next.js Mastery

> "The React Framework for the Web"

---

## What is Next.js?

Next.js is a **React framework** that provides the infrastructure to build full-stack web applications. Created by Vercel, it has become the dominant way to build React applications.

```
┌─────────────────────────────────────────────────────────────┐
│                    NEXT.JS EVOLUTION                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   2016: Next.js 1.0                                         │
│         └─ Server-side rendering for React                  │
│                                                              │
│   2020: Next.js 9.3                                         │
│         └─ getStaticProps, getServerSideProps               │
│                                                              │
│   2022: Next.js 13 (App Router)                             │
│         └─ React Server Components                          │
│         └─ Nested layouts                                   │
│         └─ Streaming                                        │
│                                                              │
│   2024: Next.js 15                                          │
│         └─ Turbopack (stable)                               │
│         └─ Partial Prerendering                             │
│         └─ Enhanced caching                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## App Router: The Modern Way

Next.js 13+ introduced the **App Router**, a complete rethinking of how we build React applications.

```
┌─────────────────────────────────────────────────────────────┐
│                    PROJECT STRUCTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   my-app/                                                    │
│   ├── app/                    # App Router (Next.js 13+)    │
│   │   ├── layout.tsx          # Root layout                 │
│   │   ├── page.tsx            # Home page (/)               │
│   │   ├── loading.tsx         # Loading UI                  │
│   │   ├── error.tsx           # Error boundary              │
│   │   ├── not-found.tsx       # 404 page                    │
│   │   ├── globals.css         # Global styles               │
│   │   │                                                     │
│   │   ├── dashboard/                                        │
│   │   │   ├── layout.tsx      # Dashboard layout            │
│   │   │   ├── page.tsx        # /dashboard                  │
│   │   │   └── settings/                                     │
│   │   │       └── page.tsx    # /dashboard/settings         │
│   │   │                                                     │
│   │   ├── blog/                                             │
│   │   │   ├── page.tsx        # /blog                       │
│   │   │   └── [slug]/         # Dynamic route               │
│   │   │       └── page.tsx    # /blog/my-post               │
│   │   │                                                     │
│   │   └── api/                # API routes                  │
│   │       └── users/                                        │
│   │           └── route.ts    # /api/users                  │
│   │                                                         │
│   ├── components/             # Shared components           │
│   ├── lib/                    # Utilities, db clients       │
│   └── public/                 # Static assets               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Server Components: The Game Changer

This is the most important concept in modern React:

```
┌─────────────────────────────────────────────────────────────┐
│              SERVER vs CLIENT COMPONENTS                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   SERVER COMPONENTS (Default)                               │
│   ────────────────────────────                              │
│   • Run on the server only                                  │
│   • Zero JavaScript sent to browser                         │
│   • Can directly access databases, filesystems              │
│   • Can use async/await at component level                  │
│   • Cannot use hooks (useState, useEffect)                  │
│   • Cannot use browser APIs (window, document)              │
│                                                              │
│   CLIENT COMPONENTS ('use client')                          │
│   ────────────────────────────────                          │
│   • Run in the browser (+ SSR for initial HTML)             │
│   • JavaScript sent to browser                              │
│   • Can use React hooks                                     │
│   • Can use browser APIs                                    │
│   • Cannot directly access server resources                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Server Component Example

```tsx
// app/posts/page.tsx
// This is a SERVER COMPONENT by default

import { db } from '@/lib/db';

// You can use async/await directly!
async function getPosts() {
  return db.query('SELECT * FROM posts ORDER BY created_at DESC');
}

export default async function PostsPage() {
  // This runs on the server
  // No API route needed
  // No useEffect, no loading state management
  const posts = await getPosts();

  return (
    <div>
      <h1>All Posts</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <a href={`/posts/${post.id}`}>{post.title}</a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Client Component Example

```tsx
// components/LikeButton.tsx
'use client'; // This directive makes it a client component

import { useState } from 'react';

export function LikeButton({ postId }: { postId: number }) {
  const [likes, setLikes] = useState(0);
  const [isLiking, setIsLiking] = useState(false);

  const handleLike = async () => {
    setIsLiking(true);
    const response = await fetch(`/api/posts/${postId}/like`, {
      method: 'POST',
    });
    const data = await response.json();
    setLikes(data.likes);
    setIsLiking(false);
  };

  return (
    <button onClick={handleLike} disabled={isLiking}>
      {isLiking ? 'Liking...' : `Like (${likes})`}
    </button>
  );
}
```

### Composing Server and Client Components

```tsx
// app/posts/[id]/page.tsx
// SERVER COMPONENT (default)

import { db } from '@/lib/db';
import { LikeButton } from '@/components/LikeButton';
import { Comments } from '@/components/Comments';

export default async function PostPage({
  params,
}: {
  params: { id: string };
}) {
  // Server-side data fetching
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, parseInt(params.id)),
  });

  if (!post) {
    notFound();
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>

      {/* Client component nested in server component */}
      <LikeButton postId={post.id} />

      {/* Another client component */}
      <Comments postId={post.id} />
    </article>
  );
}
```

```
┌─────────────────────────────────────────┐
│ 💡 TIP                                  │
│                                         │
│ Rule of thumb:                          │
│                                         │
│ • Start with Server Components          │
│ • Only add 'use client' when you need:  │
│   - useState, useEffect, useReducer     │
│   - Event handlers (onClick, onChange)  │
│   - Browser-only APIs                   │
│   - Third-party client libraries        │
│                                         │
│ Server Components = less JavaScript     │
│                   = faster page loads   │
└─────────────────────────────────────────┘
```

---

## Data Fetching Patterns

### Pattern 1: Direct Database Access (Server Components)

```tsx
// app/dashboard/page.tsx
import { db } from '@/lib/db';

export default async function Dashboard() {
  // Direct database query - no API route needed!
  const stats = await db.query(`
    SELECT
      COUNT(*) as total_users,
      COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '7 days') as new_users
    FROM users
  `);

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Total users: {stats.total_users}</p>
      <p>New this week: {stats.new_users}</p>
    </div>
  );
}
```

### Pattern 2: Server Actions (Form Submissions)

```tsx
// app/contact/page.tsx
import { db } from '@/lib/db';
import { redirect } from 'next/navigation';

// Server Action - runs on server, callable from client
async function submitContact(formData: FormData) {
  'use server';

  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  const message = formData.get('message') as string;

  await db.insert(contacts).values({ name, email, message });

  redirect('/contact/success');
}

export default function ContactPage() {
  return (
    <form action={submitContact}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />
      <textarea name="message" placeholder="Message" required />
      <button type="submit">Send</button>
    </form>
  );
}
```

### Pattern 3: API Routes (When You Need Them)

```tsx
// app/api/posts/route.ts
import { db } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const page = parseInt(searchParams.get('page') ?? '1');
  const limit = 10;

  const posts = await db.query.posts.findMany({
    limit,
    offset: (page - 1) * limit,
    orderBy: desc(posts.createdAt),
  });

  return NextResponse.json(posts);
}

export async function POST(request: Request) {
  const body = await request.json();

  const post = await db.insert(posts).values({
    title: body.title,
    content: body.content,
  }).returning();

  return NextResponse.json(post, { status: 201 });
}
```

---

## Layouts and Templates

### Root Layout (Required)

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My App',
  description: 'Built with Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <header>
          <nav>{/* Navigation */}</nav>
        </header>
        <main>{children}</main>
        <footer>{/* Footer */}</footer>
      </body>
    </html>
  );
}
```

### Nested Layouts

```tsx
// app/dashboard/layout.tsx
import { Sidebar } from '@/components/Sidebar';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <Sidebar />
      <div className="flex-1">{children}</div>
    </div>
  );
}
```

```
┌─────────────────────────────────────────────────────────────┐
│                    LAYOUT NESTING                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   URL: /dashboard/settings                                   │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  RootLayout (app/layout.tsx)                        │   │
│   │  ┌─────────────────────────────────────────────┐   │   │
│   │  │  DashboardLayout (app/dashboard/layout.tsx) │   │   │
│   │  │  ┌───────────────────────────────────────┐ │   │   │
│   │  │  │  SettingsPage                         │ │   │   │
│   │  │  │  (app/dashboard/settings/page.tsx)    │ │   │   │
│   │  │  └───────────────────────────────────────┘ │   │   │
│   │  └─────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   Layouts persist across navigations!                       │
│   Only the page component re-renders.                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Loading and Error States

### Loading UI

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="flex items-center justify-center h-64">
      <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500" />
    </div>
  );
}
```

### Error Handling

```tsx
// app/dashboard/error.tsx
'use client'; // Error components must be client components

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center justify-center h-64">
      <h2>Something went wrong!</h2>
      <p className="text-gray-500">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        Try again
      </button>
    </div>
  );
}
```

### Not Found

```tsx
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center h-64">
      <h2>Page Not Found</h2>
      <p>Could not find the requested resource</p>
      <Link href="/" className="mt-4 text-blue-500">
        Return Home
      </Link>
    </div>
  );
}
```

---

## Dynamic Routes

### Basic Dynamic Routes

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
}

export default async function BlogPost({ params }: Props) {
  const post = await getPostBySlug(params.slug);

  if (!post) {
    notFound();
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}

// Generate static pages at build time
export async function generateStaticParams() {
  const posts = await getAllPosts();

  return posts.map((post) => ({
    slug: post.slug,
  }));
}
```

### Catch-all Routes

```tsx
// app/docs/[...slug]/page.tsx
// Matches: /docs/intro, /docs/guides/getting-started, etc.

interface Props {
  params: { slug: string[] };
}

export default function DocsPage({ params }: Props) {
  // params.slug = ['guides', 'getting-started'] for /docs/guides/getting-started
  const path = params.slug.join('/');

  return <div>Docs: {path}</div>;
}
```

---

## Caching and Revalidation

```
┌─────────────────────────────────────────────────────────────┐
│                    CACHING STRATEGIES                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STATIC (Default for GET without dynamic data)              │
│  ─────────────────────────────────────────────              │
│  • Rendered at build time                                   │
│  • Cached on CDN                                            │
│  • Fastest possible response                                │
│                                                              │
│  export default async function Page() {                     │
│    const data = await fetch('...'); // Cached by default   │
│    return <div>{data}</div>;                                │
│  }                                                          │
│                                                              │
│  ─────────────────────────────────────────────              │
│                                                              │
│  REVALIDATE (ISR - Incremental Static Regeneration)         │
│  ─────────────────────────────────────────────              │
│  • Regenerate page after specified time                     │
│                                                              │
│  export const revalidate = 60; // Revalidate every 60s      │
│                                                              │
│  export default async function Page() {                     │
│    const data = await fetch('...');                         │
│    return <div>{data}</div>;                                │
│  }                                                          │
│                                                              │
│  ─────────────────────────────────────────────              │
│                                                              │
│  DYNAMIC (No caching)                                       │
│  ─────────────────────────────────────────────              │
│  • Re-render on every request                               │
│                                                              │
│  export const dynamic = 'force-dynamic';                    │
│                                                              │
│  // Or use dynamic functions:                               │
│  // cookies(), headers(), searchParams                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### On-Demand Revalidation

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request: Request) {
  const { path, tag, secret } = await request.json();

  // Verify secret
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 });
  }

  if (path) {
    revalidatePath(path); // Revalidate specific path
  }

  if (tag) {
    revalidateTag(tag); // Revalidate by cache tag
  }

  return Response.json({ revalidated: true });
}
```

---

## Middleware

```typescript
// middleware.ts (at project root)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Check auth
  const token = request.cookies.get('session');

  // Redirect unauthenticated users
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Add headers
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'hello');

  // Geolocation-based routing
  const country = request.geo?.country ?? 'US';
  if (country === 'BR' && request.nextUrl.pathname === '/') {
    return NextResponse.redirect(new URL('/br', request.url));
  }

  return response;
}

// Only run on specific paths
export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*', '/'],
};
```

---

## Common Patterns

### Authentication with Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import { verifyToken } from '@/lib/auth';

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('token')?.value;

  // Public routes
  const publicPaths = ['/login', '/register', '/api/auth'];
  if (publicPaths.some(path => request.nextUrl.pathname.startsWith(path))) {
    return NextResponse.next();
  }

  // Protected routes
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    const user = await verifyToken(token);
    const response = NextResponse.next();
    response.headers.set('x-user-id', user.id);
    return response;
  } catch {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

### Data Fetching with Loading States

```tsx
// app/users/page.tsx
import { Suspense } from 'react';
import { UserList } from './UserList';
import { UserListSkeleton } from './UserListSkeleton';

export default function UsersPage() {
  return (
    <div>
      <h1>Users</h1>
      <Suspense fallback={<UserListSkeleton />}>
        <UserList />
      </Suspense>
    </div>
  );
}

// UserList.tsx (Server Component)
async function UserList() {
  const users = await getUsers(); // Slow fetch
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

## Exercise: Build a Blog with Next.js

```
┌─────────────────────────────────────────┐
│ 🎯 EXERCISE                             │
│                                         │
│ Create a simple blog:                   │
│                                         │
│ 1. Create new Next.js app:              │
│    npx create-next-app@latest my-blog   │
│    --typescript --tailwind --app        │
│                                         │
│ 2. Create these routes:                 │
│    /              → Home with post list │
│    /posts/[slug]  → Individual post     │
│    /about         → About page          │
│                                         │
│ 3. Create a layout with navigation      │
│                                         │
│ 4. Add loading.tsx for posts list       │
│                                         │
│ 5. Add error.tsx for error handling     │
│                                         │
│ 6. Create a Server Component that       │
│    fetches posts (mock data is fine)    │
│                                         │
│ 7. Create a Client Component for        │
│    a "like" button with useState        │
│                                         │
│ BONUS: Add ISR with revalidate = 60     │
└─────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Server Components are the default** — Start there, add 'use client' only when needed
2. **Direct database access is possible** — No API routes required for read operations
3. **Server Actions simplify forms** — No separate API endpoints for mutations
4. **Layouts persist across navigation** — Great for performance
5. **Caching is powerful** — ISR gives you the best of static and dynamic

---

## What's Next?

You now have the tools to build modern web applications. In Chapter 7, we'll explore **Claude Code** — your AI pair programmer that can help you build faster and better.

---

[← Previous: Express to Fastify](05-express-to-fastify.md) | [Back to Contents](../README.md) | [Next: Claude Code →](07-claude-code.md)
