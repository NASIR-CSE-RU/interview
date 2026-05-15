# Next.js App Router — Interview Preparation Guide

---

## 1. File-Based Routing & Dynamic Routes

### Concepts
Next.js uses the **filesystem as the router**. Every `page.tsx` inside `app/` becomes a route automatically.

| File | Route |
|---|---|
| `app/page.tsx` | `/` |
| `app/about/page.tsx` | `/about` |
| `app/blog/[slug]/page.tsx` | `/blog/:slug` |
| `app/blog/[...slug]/page.tsx` | `/blog/*` (catch-all) |
| `app/blog/[[...slug]]/page.tsx` | `/blog` + `/blog/*` (optional catch-all) |

### Code Example

```tsx
// app/blog/[slug]/page.tsx
type Props = { params: { slug: string } };

export default function BlogPost({ params }: Props) {
  return <h1>Post: {params.slug}</h1>;
}

// Generate static paths at build time
export async function generateStaticParams() {
  const posts = await fetch("/api/posts").then((r) => r.json());
  return posts.map((p: { slug: string }) => ({ slug: p.slug }));
}
```

### Interview Q&A

**Q: What is the difference between `[slug]`, `[...slug]`, and `[[...slug]]`?**

- `[slug]` — matches a **single** segment: `/blog/hello`
- `[...slug]` — matches **one or more** segments: `/blog/2024/jan/hello` → `slug = ['2024','jan','hello']`
- `[[...slug]]` — **optional** catch-all; also matches the base route with no segments

**Q: How do you access route params in a Server Component?**

Params are passed as a `params` prop. In Next.js 15+, `params` is a **Promise** and must be awaited: `const { slug } = await params`.

---

## 2. Server Actions & Mutations

### Concepts
Server Actions are **async functions that run on the server**, triggered from the client (forms, buttons). Defined with `"use server"` directive.

```tsx
// app/actions.ts
"use server";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  await db.post.create({ data: { title } });
  revalidatePath("/blog");
}
```

```tsx
// app/new-post/page.tsx
import { createPost } from "../actions";

export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Post title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

### Interview Q&A

**Q: Where can Server Actions be defined?**

In a separate file with `"use server"` at the top, or inline inside a Server Component with `"use server"` at the top of the function body.

**Q: How do you handle optimistic UI with Server Actions?**

Use the `useOptimistic` hook to show a temporary UI state before the server responds, then let the actual result replace it.

**Q: What does `revalidatePath` / `revalidateTag` do?**

They **purge the cache** for a given path or cache tag so the next request fetches fresh data.

---

## 3. Middleware (`middleware.ts`)

### Concepts
Middleware runs **before a request is processed** — on every matched route, at the Edge. Used for auth, redirects, A/B testing, locale detection.

```ts
// middleware.ts (at project root)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("auth-token");

  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

### Interview Q&A

**Q: Where does middleware run?**

At the **Edge runtime** — before the request hits your server components or route handlers. It's extremely fast but has a limited runtime (no Node.js APIs).

**Q: What is the `matcher` config?**

An array of path patterns that tells Next.js which routes the middleware should run on. Without it, middleware runs on every request.

**Q: Can middleware modify the response body?**

No. It can only **rewrite**, **redirect**, **set headers/cookies**, or pass the request through with `NextResponse.next()`.

---

## 4. Route Handlers (API Routes)

### Concepts
Replace `pages/api/` files. Defined as `route.ts` inside `app/`, exporting named HTTP method functions.

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const posts = await db.post.findMany();
  return NextResponse.json(posts);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const post = await db.post.create({ data: body });
  return NextResponse.json(post, { status: 201 });
}
```

```ts
// app/api/posts/[id]/route.ts
export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await db.post.delete({ where: { id: params.id } });
  return new NextResponse(null, { status: 204 });
}
```

### Interview Q&A

**Q: How are route handlers different from Server Actions?**

Route handlers are traditional **REST endpoints** (used by external clients, mobile apps, curl). Server Actions are **React-integrated mutations** — called directly from components without a manual `fetch`.

**Q: Can a route.ts and page.tsx coexist in the same folder?**

No. A segment can either be a UI route (`page.tsx`) or an API route (`route.ts`), not both.

---

## 5. Parallel & Intercepting Routes

### Concepts

**Parallel Routes** render multiple pages in the same layout simultaneously using **named slots** (`@slot`).

```
app/dashboard/
  layout.tsx         ← receives @team and @analytics as props
  @team/page.tsx
  @analytics/page.tsx
  page.tsx
```

```tsx
// app/dashboard/layout.tsx
export default function Layout({ team, analytics }: {
  team: React.ReactNode;
  analytics: React.ReactNode;
}) {
  return (
    <div>
      {team}
      {analytics}
    </div>
  );
}
```

**Intercepting Routes** let you load a route inside the current layout (e.g., open a photo in a modal without leaving the feed). Uses `(.)`, `(..)`, `(...)` conventions.

```
app/feed/
  page.tsx
  (..)photo/[id]/page.tsx   ← intercepts /photo/[id] when navigating from feed
```

### Interview Q&A

**Q: What is a real-world use case for parallel routes?**

A dashboard with an independent sidebar and main content area — each can have its own loading state, error boundary, and streaming.

**Q: What does `(..)photo` mean in intercepting routes?**

`(..)` means go **one level up** in the route hierarchy to match `/photo/[id]`. The conventions mirror relative file paths: `(.)` = same level, `(..)` = one up, `(...)` = root.

---

## 6. `next/image`, `next/font`, `next/link` Optimizations

### `next/image`
- Auto **WebP/AVIF** conversion, lazy loading, prevents layout shift (requires `width`/`height` or `fill`)
- `priority` prop for LCP images (above the fold)

```tsx
import Image from "next/image";

<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority />
```

### `next/font`
- Self-hosts fonts at build time — **zero layout shift**, no external network requests

```tsx
import { Geist } from "next/font/google";

const geist = Geist({ subsets: ["latin"], variable: "--font-geist" });

export default function Layout({ children }: { children: React.ReactNode }) {
  return <html className={geist.variable}>{children}</html>;
}
```

### `next/link`
- Prefetches linked pages automatically in the viewport
- No full page reload; uses client-side navigation

```tsx
import Link from "next/link";

<Link href="/blog/hello">Read post</Link>
```

### Interview Q&A

**Q: Why use `next/image` instead of `<img>`?**

Automatic format optimization, responsive sizes, lazy loading by default, and prevents Cumulative Layout Shift (CLS).

**Q: What problem does `next/font` solve?**

Eliminates **flash of unstyled text (FOUT)** and removes the privacy/performance cost of loading fonts from Google's CDN at runtime.

---

## 7. Caching Layers

Next.js has **four distinct caching layers**:

| Cache | What it stores | Duration | Invalidated by |
|---|---|---|---|
| **Request Memoization** | `fetch` results within a single render tree | Per request (in-memory) | Automatic — per request |
| **Data Cache** | `fetch` responses on the server (persistent) | Indefinitely (until revalidated) | `revalidatePath`, `revalidateTag`, `cache: 'no-store'` |
| **Full Route Cache** | Statically rendered HTML + RSC payload | Until revalidation/redeploy | Redeployment or on-demand revalidation |
| **Router Cache** | RSC payload cached in the browser | Session (30s / 5min depending on type) | `router.refresh()`, Server Actions |

### Code Example — Controlling Data Cache

```ts
// Cache forever (default static)
fetch("/api/data");

// Cache with revalidation every 60 seconds
fetch("/api/data", { next: { revalidate: 60 } });

// No cache — always fetch fresh
fetch("/api/data", { cache: "no-store" });

// Tag for targeted invalidation
fetch("/api/posts", { next: { tags: ["posts"] } });
```

### Interview Q&A

**Q: What is Request Memoization?**

If you call the same `fetch` URL multiple times during a single server render (e.g., in a layout AND a page), Next.js **deduplicates** them — only one actual HTTP request is made.

**Q: What's the difference between Data Cache and Full Route Cache?**

- **Data Cache** = caches the raw fetch response (data layer)
- **Full Route Cache** = caches the entire rendered HTML/RSC payload of a route (output layer)

**Q: How do you opt a route into dynamic rendering?**

Use `export const dynamic = "force-dynamic"` or add `cache: 'no-store'` to any fetch — Next.js will skip the Full Route Cache.

---

## 8. Metadata API for SEO

### Static Metadata

```tsx
// app/blog/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Blog",
  description: "Read our latest posts",
  openGraph: {
    title: "Blog",
    description: "Read our latest posts",
    images: ["/og-image.png"],
  },
};
```

### Dynamic Metadata

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from "next";

export async function generateMetadata(
  { params }: { params: { slug: string } }
): Promise<Metadata> {
  const post = await fetch(`/api/posts/${params.slug}`).then((r) => r.json());

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [post.coverImage],
    },
  };
}
```

### Title Templates

```tsx
// app/layout.tsx
export const metadata: Metadata = {
  title: {
    template: "%s | My Blog",
    default: "My Blog",
  },
};

// app/about/page.tsx — renders as "About | My Blog"
export const metadata: Metadata = { title: "About" };
```

### Interview Q&A

**Q: How does `generateMetadata` differ from the `metadata` export?**

`metadata` is **static** — defined at build time. `generateMetadata` is **async** and can fetch data to produce dynamic titles, descriptions, and OG images per page.

**Q: Can metadata be defined in `layout.tsx`?**

Yes. Metadata is **merged** from layout → page. The page-level metadata overrides the layout's values for the same fields.

**Q: How do you generate dynamic Open Graph images?**

Use the `ImageResponse` API via a special `opengraph-image.tsx` file in the route segment:

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from "next/og";

export default async function OGImage({ params }: { params: { slug: string } }) {
  return new ImageResponse(
    <div style={{ fontSize: 48, background: "white", padding: 40 }}>
      {params.slug}
    </div>,
    { width: 1200, height: 630 }
  );
}
```

---

## Quick-Reference Cheat Sheet

```
Routing
  [slug]            → single dynamic segment
  [...slug]         → catch-all (one or more)
  [[...slug]]       → optional catch-all
  @slot             → parallel route slot
  (.)/(..)/(...) → intercepting route levels

Caching
  fetch()                          → Data Cache (static)
  fetch({cache:'no-store'})        → skip Data Cache
  fetch({next:{revalidate:N}})     → ISR-style revalidation
  fetch({next:{tags:['x']}})       → tag for revalidateTag('x')
  revalidatePath('/path')          → purge by path
  revalidateTag('x')               → purge by tag
  dynamic = 'force-dynamic'        → always dynamic route

Server Actions
  'use server'  → marks function as Server Action
  revalidatePath / revalidateTag inside actions → refresh cache after mutation

Metadata
  export const metadata            → static
  export async function generateMetadata → dynamic
  title.template: '%s | Site'      → title inheritance
  opengraph-image.tsx              → dynamic OG image
```