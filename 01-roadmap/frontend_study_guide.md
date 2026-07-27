# Next.js & React Frontend Architecture Study Guide

This guide covers advanced React architecture, Next.js App Router internals, state management paradigms, performance profiling, testing patterns, and security best practices for senior engineering roles.

---

## 1. React Core Architecture & Fiber Engine

### React Fiber Architecture
* **The Fiber Engine:** Fiber is React’s execution engine. It replaced the synchronous stack reconciler. Fiber structures components as a linked list of virtual frames (Fibers).
* **Incremental Rendering:** Allows React to break rendering work into chunks, pause execution, yield back to the main browser thread to process events, and resume rendering later. This keeps pages responsive under heavy UI updates.
* **Two Phases of Reconciliation:**
  1. **Render Phase:** Asynchronous and interruptible. React builds a new virtual tree of elements, computes updates, and flags changes (effects).
  2. **Commit Phase:** Synchronous and uninterruptible. React applies changes (insertions, updates, deletions) directly to the DOM in a single pass.

### React Hooks Rules & Lifecycle
* **Rules of Hooks:** Hooks must be called at the top level of a functional component (not inside loops, conditions, or nested functions). Hooks rely on a stable execution order to map values to their corresponding fiber nodes.
* **Custom Hooks:** Used to extract stateful component logic into reusable functions (e.g. `useAuth`, `useLocalStorage`).

---

## 2. Next.js App Router vs. Pages Router

### Next.js App Router (Next.js 13.4+)
* **Directory Structure:** Uses a file-system based router based on directories under the `app/` folder. Pages are defined by a `page.tsx` file inside route folders.
* **React Server Components (RSC) by Default:** Components in the App Router are Server Components by default unless marked with `'use client'`.
* **Layouts and Nesting:** Supports shared layouts (`layout.tsx`) that do not re-render during navigation, preserving client-side state.

### React Server Components (RSC) vs. Client Components

```
┌─────────────────────────────────────────┬─────────────────────────────────────────┐
│ React Server Components (RSC)           │ Client Components ('use client')        │
├─────────────────────────────────────────┼─────────────────────────────────────────┤
│ - Rendered on the server.               │ - Hydrated in the browser.              │
│ - Send zero JS to the browser bundle.   │ - Add JS overhead to client bundle.     │
│ - Can query databases/APIs directly.    │ - Cannot query server resources directly│
│ - Cannot use hooks (useState, useEffect)│ - Can use hooks, state, and browser APIs│
└─────────────────────────────────────────┴─────────────────────────────────────────┘
```

* **Hydration Process:** The server pre-renders Client Components to raw HTML. In the browser, React downloads the associated JavaScript bundle, hooks up event listeners, and reconstructs the active state in memory (hydration).

---

## 3. Data Fetching, Caching, & Server Actions

### Next.js Data Fetching Strategies
* **Static Rendering (equivalent to SSG):** HTML is generated at build time. Best for static documentation or landing pages.
* **Dynamic Rendering (equivalent to SSR):** HTML is generated on the server for every incoming request. Best for user dashboards.
* **Incremental Static Regeneration (ISR):** Generates static pages at build time but updates them in the background after a specified duration using:
  ```typescript
  fetch('https://api.example.com/data', { next: { revalidate: 60 } }) // revalidate every 60 seconds
  ```

### Server Actions (React 19 / Next.js 14)
Server Actions are asynchronous functions declared with `'use server'` that execute on the server. They allow developers to handle form submissions and database mutations without manually writing API routes:
```typescript
// app/actions.ts
'use server'

import { revalidatePath } from 'next/cache';

public async void createOrder(formData: FormData) {
    const orderData = {
        item: formData.get('item'),
        quantity: Number(formData.get('quantity')),
    };
    await db.save(orderData);
    revalidatePath('/orders'); // Clears the server-side cache for this page
}
```

---

## 4. State Management Paradigm for Enterprise SaaS

Enterprise B2B SaaS platforms (such as IM's dashboards) should separate **Server State** (data fetched from APIs) from **Client UI State** (UI elements like sidebar toggles, theme configurations, active tabs):

```
                                  Global State
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
     [ Server State ]                                      [ Client State ]
     - Handles DB/API data caching                         - Handles transient UI state
     - Polling, pagination cache                           - Small, local updates
     - TanStack / React Query                              - Zustand / Context API
```

1. **Server State (TanStack Query / React Query):**
   * *Why:* Automatically handles caching, query deduplication, background revalidation, cache invalidation, and optimistic updates.
2. **Client State (Zustand):**
   * *Why:* Extremely lightweight (1KB), boilerplate-free, does not require wrapping components in Context Providers (preventing global re-renders on every state update).
3. **Avoid Redux:** Redux adds high boilerplate overhead and is unnecessary when Server State is managed by TanStack Query.

---

## 5. Performance Optimization & Core Web Vitals

To hit production performance targets (e.g. INP < 200ms, LCP < 2.5s):

### Bundle Size Control
* **Code Splitting / Dynamic Imports:** Defer loading non-critical components (such as heavy charts, modals, or PDF export libraries) until they are needed:
  ```typescript
  import dynamic from 'next/dynamic';
  const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
      loading: () => <SkeletonLoader />,
  });
  ```
* **Tree Shaking:** Ensure imports are structured so unused code is omitted during build bundling. Avoid importing entire libraries if only a single function is needed.

### Core Web Vitals Profiling
* **LCP (Largest Contentful Paint):** Pre-render hero text and main visual elements on the server. Use Next.js `next/image` to automatically resize, compress, and serve WebP images. Mark above-the-fold images with `priority`.
* **INP (Interaction to Next Paint):** Prevent long-running JavaScript execution blocks (> 50ms) from blocking the main thread. Break long operations up using React's `useTransition` hook to mark rendering updates as non-blocking.
* **CLS (Cumulative Layout Shift):** Ensure skeleton loaders have fixed layout dimensions that match the final fetched dashboard elements to prevent page elements from shifting during loading.

---

## 6. Frontend Testing & Security

### Testing Strategy
* **Unit Testing:** Use **Jest** and **React Testing Library** to test utility functions, custom hooks, and isolated components.
* **API Mocking (MSW - Mock Service Worker):** Intercepts network queries at the network level, simulating server responses for tests without hitting real API servers.
* **E2E Testing:** Use **Cypress** or **Playwright** to validate critical user flows (e.g. login, payment integration).

### Security
* **Cross-Site Scripting (XSS):** React escapes strings by default. However, avoid using `dangerouslySetInnerHTML` unless input is sanitized first using libraries like DOMPurify.
* **Cross-Site Request Forgery (CSRF):** Use double-submit CSRF tokens or secure cookies with the `SameSite=Strict` flag.
* **Authentication Storage:** Do not store access tokens in `localStorage` or `sessionStorage` (vulnerable to XSS extraction). Store tokens in memory, or use secure, HTTP-only, `SameSite=Strict` cookies.

---

### Questions & Answers: Frontend Architecture

#### Q1: Explain React's Reconciliation and the Virtual DOM. How does React determine which elements to update?
**Answer:**
> "The Virtual DOM is an in-memory representation of the real DOM. When state changes, React runs the render phase, building a new Virtual DOM tree.
> React determines differences using the **Reconciliation Algorithm** (Diffing):
> 1. **Element Type:** If two elements have different types (e.g. changing a `<div>` to a `<span>`), React tears down the old tree and builds the new one from scratch.
> 2. **Keys in Lists:** React uses the `key` prop to identify elements in collections across renders. If a key is missing or is just an array index, React may re-render or shift list items incorrectly.
> 3. **Component Identity:** If elements are identical, React only updates changed attributes (such as class names or text contents) without replacing the underlying DOM node."

#### Q2: Write a Next.js Client Component that fetches a list of products using TanStack Query, showing loading and error states.
```typescript
'use client'

import { useQuery } from '@tanstack/react-query';

interface Product {
    id: string;
    name: string;
    price: number;
}

async function fetchProducts(): Promise<Product[]> {
    const response = await fetch('/api/products');
    if (!response.ok) {
        throw new Error('Network error');
    }
    return response.json();
}

public async void ProductList() {
    const { data: products, isLoading, error } = useQuery<Product[]>({
        queryKey: ['products'],
        queryFn: fetchProducts,
    });

    if (isLoading) return <div>Loading products...</div>;
    if (error) return <div>Error loading data: {error.message}</div>;

    return (
        <ul>
            {products?.map(product => (
                <li key={product.id}>
                    {product.name} - ${product.price}
                </li>
            ))}
        </ul>
    );
}
```

---
Frontend Architecture Study Guide
