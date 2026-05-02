# 🚀 Senior Frontend Developer (Next.js) - Interview Ready Materials

A comprehensive guide to help you prepare for a Senior Frontend Developer interview focused on **Next.js**.

---

## 📚 1. Core Next.js Concepts to Master

### Rendering Strategies
- **SSR (Server-Side Rendering)** – Page rendered on each request.
- **SSG (Static Site Generation)** – Pre-rendered at build time.
- **ISR (Incremental Static Regeneration)** – Static + revalidation.
- **CSR (Client-Side Rendering)** – Rendered in the browser.
- **RSC (React Server Components)** – New paradigm in App Router.

### App Router vs Pages Router

| Feature | Pages Router | App Router |
|---------|-------------|------------|
| File convention | `pages/` | `app/` |
| Data fetching | `getServerSideProps`, `getStaticProps` | `async` components, `fetch` |
| Layouts | Custom `_app.js` | Nested `layout.js` |
| Server Components | ❌ | ✅ |
| Streaming | Limited | ✅ Built-in |
| Loading states | Manual | `loading.js` |

### Key Features to Know
- File-based routing & dynamic routes (`[slug]`, `[...slug]`, `[[...slug]]`)
- Server Actions & mutations
- Middleware (`middleware.ts`)
- Route handlers (API routes)
- Parallel & intercepting routes
- `next/image`, `next/font`, `next/link` optimizations
- Caching layers (Request Memoization, Data Cache, Full Route Cache, Router Cache)
- Metadata API for SEO

---

## 💼 2. Technical Interview Questions & Answers

### 🟢 Beginner to Intermediate

**Q1. What is Next.js and how does it differ from React?**  
React is a UI library for building components. Next.js is a full-stack React framework providing SSR, SSG, file-based routing, API routes, image optimization, and performance optimizations out of the box.

**Q2. Explain `getStaticProps`, `getServerSideProps`, and `getStaticPaths`.**  
- `getStaticProps` – Fetches data at build time (SSG).  
- `getServerSideProps` – Fetches data on each request (SSR).  
- `getStaticPaths` – Defines dynamic routes to pre-render statically.

**Q3. What is ISR and when would you use it?**  
ISR allows static pages to be regenerated after deployment using `revalidate`. Useful for content that updates periodically (e.g., blogs, product pages).

**Q4. What is the difference between `<Link>` and `<a>` in Next.js?**  
`<Link>` enables client-side navigation with prefetching, while `<a>` causes a full-page reload.

**Q5. How does `next/image` improve performance?**  
Provides automatic resizing, lazy loading, modern format conversion (AVIF/WebP), and CDN-optimized delivery.

---

### 🟡 Intermediate to Advanced

**Q6. Explain Server Components vs Client Components.**  
- **Server Components** – Render on server, no JS sent to client, can access DB/secrets directly.  
- **Client Components** – Need `"use client"`, support hooks/state/events.

**Q7. What are Server Actions?**  
Async functions that run on the server, callable from client components, typically used for form submissions and mutations without explicit API endpoints.

**Q8. How does caching work in the App Router?**  
Four layers:  
1. **Request Memoization** – Per-render fetch deduplication.  
2. **Data Cache** – Persistent across requests.  
3. **Full Route Cache** – Stores rendered RSC payload.  
4. **Router Cache** – Client-side navigation cache.

**Q9. What is Middleware in Next.js?**  
Code that runs before a request completes. Use cases: authentication, redirects, A/B testing, geo-localization, rewrites.

**Q10. Explain Parallel and Intercepting Routes.**  
- **Parallel Routes** – Render multiple pages simultaneously (`@slot` folders).  
- **Intercepting Routes** – Show route as overlay/modal while preserving URL (`(.)`, `(..)`).

**Q11. How would you optimize a slow Next.js application?**  
- Use RSC where possible.  
- Lazy-load with `next/dynamic`.  
- Optimize images & fonts.  
- Use proper caching/revalidation.  
- Audit bundle with `@next/bundle-analyzer`.  
- Code-split, prefetch routes, tree-shake.

**Q12. How do you handle authentication in Next.js?**  
- NextAuth.js / Auth.js, Clerk, Supabase, or custom JWT.  
- Use middleware for route protection.  
- Use cookies (HttpOnly) for session storage.

---

### 🔴 Senior-Level / System Design

**Q13. How would you architect a large-scale Next.js app?**  
- Feature-based folder structure.  
- Shared `ui` and `lib` packages (monorepo via Turborepo).  
- Separation of concerns: server vs client, business vs UI.  
- Strict TypeScript + ESLint + Prettier.  
- CI/CD with preview deploys (Vercel/Netlify).  
- Observability: Sentry, Datadog, OpenTelemetry.

**Q14. How do you handle SEO in Next.js?**  
- Metadata API (`generateMetadata`).  
- Structured data (JSON-LD).  
- Server-rendered content for crawlers.  
- Sitemaps and `robots.txt`.  
- Open Graph & Twitter card tags.

**Q15. How do you ensure accessibility (a11y) in Next.js apps?**  
- Semantic HTML.  
- ARIA attributes where needed.  
- Keyboard navigation testing.  
- Tools: `eslint-plugin-jsx-a11y`, Axe, Lighthouse.

**Q16. How do you manage state in a large Next.js application?**  
- Server state: React Query / SWR / RSC.  
- Client state: Zustand, Redux Toolkit, Jotai.  
- Form state: React Hook Form + Zod.  
- URL state: search params + `useSearchParams`.

**Q17. How would you migrate a Pages Router project to App Router?**  
- Incremental migration (both can coexist).  
- Move shared layouts to `app/layout.tsx`.  
- Replace data fetching with async components.  
- Convert API routes to Route Handlers.  
- Test progressively per feature.

**Q18. Explain Streaming and Suspense in Next.js.**  
Streaming sends HTML chunks as they're ready. Combined with `<Suspense>`, it allows progressive rendering and reduced TTFB.

---

## 🧠 3. JavaScript / TypeScript / React Fundamentals

- Closures, hoisting, event loop, microtasks vs macrotasks.
- `var` vs `let` vs `const`, scoping.
- Promises, async/await, error handling.
- TypeScript: generics, utility types (`Partial`, `Pick`, `Omit`, `Record`).
- React hooks: `useState`, `useEffect`, `useMemo`, `useCallback`, `useRef`, `useReducer`, `useContext`, `useTransition`, `useDeferredValue`.
- Reconciliation, virtual DOM, keys, memoization.
- Custom hooks design patterns.

---

## 🛠️ 4. Practical Coding Tasks (Practice These)

1. Build a paginated product list with ISR.  
2. Implement search with debounced URL params.  
3. Create a protected dashboard using middleware + JWT.  
4. Build a modal using intercepting routes.  
5. Implement infinite scroll with Server Actions.  
6. Optimize a slow page (provide before/after).  
7. Build a multi-step form with Server Actions + Zod validation.  
8. Implement dark mode with `next-themes`.  
9. Write unit tests with Jest + React Testing Library.  
10. Set up E2E tests with Playwright/Cypress.

---

## 🔧 5. Tools & Ecosystem

| Category | Tools |
|----------|-------|
| Styling | Tailwind CSS, CSS Modules, styled-components |
| UI Libraries | shadcn/ui, Radix UI, MUI, Chakra |
| State Mgmt | Zustand, Redux Toolkit, Jotai |
| Data Fetching | TanStack Query, SWR |
| Forms | React Hook Form, Zod, Yup |
| Testing | Jest, RTL, Playwright, Cypress, Vitest |
| Auth | NextAuth.js, Clerk, Auth0 |
| CMS | Sanity, Contentful, Strapi |
| Deployment | Vercel, AWS Amplify, Netlify, Docker |
| Monitoring | Sentry, Datadog, LogRocket |

---

## 🎯 6. Behavioral Interview Questions (STAR Method)

1. Tell me about a challenging Next.js project and how you delivered it.  
2. Describe a time you mentored junior developers.  
3. Tell me about a production bug you fixed under pressure.  
4. How do you handle disagreements with backend engineers or designers?  
5. Describe a trade-off you made between performance and developer experience.  
6. How do you stay updated with the rapidly changing JS ecosystem?  
7. Tell me about a time you led a migration or architectural change.

**Use the STAR framework:**  
- **S**ituation – Set the context.  
- **T**ask – Define your responsibility.  
- **A**ction – Describe what you did.  
- **R**esult – Share measurable outcomes.

---

## 📊 7. System Design Topics for Frontend

- Designing a design system / component library.  
- Micro-frontend architecture.  
- Real-time apps (WebSockets, SSE).  
- Internationalization (i18n) at scale.  
- Performance budgets & Core Web Vitals.  
- Multi-tenant SaaS frontend architecture.  
- Edge rendering vs Node runtime.  
- Feature flags and A/B testing.

---

## ✅ 8. Final Checklist Before the Interview

- [ ] Review your past Next.js projects with metrics.  
- [ ] Be ready to whiteboard component architecture.  
- [ ] Practice live coding (CodeSandbox / StackBlitz).  
- [ ] Prepare 2–3 thoughtful questions for the interviewer.  
- [ ] Read the company's blog/engineering posts.  
- [ ] Refresh on Web Vitals (LCP, FID/INP, CLS).  
- [ ] Review the latest Next.js release notes.  
- [ ] Prepare your GitHub/portfolio for review.

---

## ❓ 9. Smart Questions to Ask the Interviewer

1. What's the team's current Next.js version and migration roadmap?  
2. How are performance and Core Web Vitals tracked in production?  
3. What does the deployment & CI/CD pipeline look like?  
4. How are design and engineering aligned on component systems?  
5. What are the biggest technical challenges the team is facing?  
6. How is technical debt managed?  
7. What does success in the first 90 days look like?

---

## 📖 10. Recommended Resources

- **Official Docs:** https://nextjs.org/docs  
- **React Docs:** https://react.dev  
- **Patterns.dev:** https://www.patterns.dev  
- **Web.dev:** https://web.dev  
- **Vercel Blog:** https://vercel.com/blog  
- **Josh Comeau's Blog:** https://www.joshwcomeau.com  
- **Lee Robinson's YouTube:** https://www.youtube.com/@leerob  

---

### 💡 Pro Tip
> "Senior" isn't only about coding ability — it's about **decision-making, trade-offs, mentorship, and impact**. Always frame your answers with business and team context, not just technical detail.

---

**Good luck with your interview! 🚀**
