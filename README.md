## Starting the Project

Students should first follow the experiment guide to proceed with the project

- [Experiment 7 Guide](https://github.com/cu-fs1#experiment-7)

# Product Manager

A Next.js 16 app that demonstrates a **persistent product list** using **Zustand** with the `persist` middleware, **shadcn/ui** components, and **Tailwind CSS v4**.

Users can view a list of products, add new ones, update quantities, and delete items. All data is persisted to `localStorage` so it survives page reloads.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Next.js 16 (App Router) | Framework |
| React 19 | UI library |
| TypeScript | Type safety |
| Tailwind CSS v4 | Styling |
| Zustand v5 | Global state management |
| `zustand/middleware` `persist` | localStorage persistence |
| shadcn/ui | Pre-built UI components (Button, Input, Spinner) |
| `clsx` + `tailwind-merge` | Conditional class merging |

---

## Project Structure

```
app/
  globals.css        # Tailwind v4 + shadcn CSS variables
  layout.tsx         # Root layout with fonts
  page.tsx           # Home page — renders <ProductList />
components/
  ProductList.tsx    # Lists all products, Add button
  ProductItem.tsx    # Single product row with Save/Delete
  ui/
    button.tsx       # shadcn Button component
    input.tsx        # shadcn Input component
    spinner.tsx      # shadcn Spinner component
store/
  useProductStore.ts # Zustand store with persist middleware
lib/
  utils.ts           # cn() helper (clsx + tailwind-merge)
```

---

## Step-by-Step Build Guide

### 1. Install dependencies

```bash
pnpm add zustand
```

### 2. Initialise shadcn/ui

```bash
pnpm dlx shadcn@latest init
```

When prompted, choose **neutral** base colour and confirm `app/globals.css` as the CSS file.

Then add the two components used in this project:

```bash
pnpm dlx shadcn@latest add button
pnpm dlx shadcn@latest add input
pnpm dlx shadcn@latest add spinner
```

This creates `components/ui/button.tsx`, `components/ui/input.tsx`, and `components/ui/spinner.tsx`.

### 4. `lib/utils.ts`

shadcn scaffolds this automatically, but for reference:

```ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

`cn()` lets you pass conditional Tailwind classes and ensures conflicting classes are resolved correctly.

### 5. `store/useProductStore.ts` — Zustand store

This is the core of the app. Create `store/useProductStore.ts`:

```ts
"use client";

import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";

export type Product = {
  id: number;
  name: string;
  price: number;
  category: string;
  quantity: number;
};

type ProductStore = {
  products: Product[];
  hydrated: boolean;
  setHydrated: (value: boolean) => void;
  addProduct: () => void;
  deleteProduct: (id: number) => void;
  updateQuantity: (id: number, quantity: number) => void;
};

const initialProducts: Product[] = [
  { id: 1, name: "Wireless Headphones", price: 79.99, category: "Electronics", quantity: 10 },
  { id: 2, name: "Running Shoes",        price: 119.99, category: "Footwear",    quantity: 5  },
  { id: 3, name: "Coffee Maker",         price: 49.99,  category: "Kitchen",     quantity: 8  },
  { id: 4, name: "Yoga Mat",             price: 29.99,  category: "Sports",      quantity: 15 },
  { id: 5, name: "Desk Lamp",            price: 34.99,  category: "Home",        quantity: 12 },
];

export const useProductStore = create<ProductStore>()(
  persist(
    (set, get) => ({
      products: initialProducts,
      hydrated: false,
      setHydrated: (value) => set({ hydrated: value }),
      addProduct: () => {
        const products = get().products;
        const nextId = products.length ? Math.max(...products.map((p) => p.id)) + 1 : 1;
        const nextIndex = products.length + 1;
        set({
          products: [
            ...products,
            { id: nextId, name: `New Product ${nextIndex}`, price: 0, category: "General", quantity: 1 },
          ],
        });
      },
      deleteProduct: (id) =>
        set((state) => ({ products: state.products.filter((p) => p.id !== id) })),
      updateQuantity: (id, quantity) =>
        set((state) => ({
          products: state.products.map((p) => (p.id === id ? { ...p, quantity } : p)),
        })),
    }),
    {
      name: "products-store",                               // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ products: state.products }), // only persist products, not hydrated flag
      onRehydrateStorage: () => (state) => {
        state?.setHydrated(true);                           // flip flag once data is loaded from storage
      },
    },
  ),
);
```

**Key concepts:**

- `persist` wraps the store creator and reads/writes to `localStorage` on every state change.
- `partialize` prevents `hydrated` and `setHydrated` from being saved to storage (they are runtime-only).
- `onRehydrateStorage` returns a callback that runs after hydration finishes — we use it to set `hydrated: true`.
- The `hydrated` flag is used in the UI to avoid a flash of stale/default content during SSR or initial mount (Next.js hydration mismatch prevention).

### 6. `components/ProductList.tsx`

```tsx
"use client";

import { useProductStore } from "../store/useProductStore";
import ProductItem from "./ProductItem";
import { Button } from "@/components/ui/button";

export default function ProductList() {
  const products = useProductStore((state) => state.products);
  const addProduct = useProductStore((state) => state.addProduct);
  const hydrated = useProductStore((state) => state.hydrated);

  if (!hydrated) return <Spinner className="size-6" />; // show spinner until localStorage is read

  return (
    <div className="w-full max-w-2xl">
      <div className="mb-6 flex items-center justify-between gap-4">
        <h1 className="text-2xl font-bold text-zinc-900 dark:text-zinc-50">
          Products{" "}
          <span className="text-base font-normal text-zinc-500">
            ({products.length})
          </span>
        </h1>
        <Button size="sm" onClick={addProduct}>Add</Button>
      </div>

      {products.length === 0 ? (
        <p className="rounded-lg border border-dashed border-zinc-300 px-6 py-10 text-center text-zinc-400 dark:border-zinc-700">
          No products left.
        </p>
      ) : (
        <ul className="flex flex-col gap-3">
          {products.map((product) => (
            <li key={product.id}>
              <ProductItem product={product} />
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Key concepts:**

- Each piece of state is selected individually with `useProductStore((state) => state.x)` — this avoids unnecessary re-renders since Zustand only re-renders when the selected slice changes.
- `if (!hydrated) return <Spinner />` prevents the component from rendering stale seed data before localStorage has been read. A spinner is shown instead of a blank screen, and Next.js hydration mismatch warnings are avoided.

### 7. `components/ProductItem.tsx`

```tsx
"use client";

import { useState } from "react";
import { useProductStore, type Product } from "../store/useProductStore";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

export default function ProductItem({ product }: { product: Product }) {
  const deleteProduct = useProductStore((state) => state.deleteProduct);
  const updateQuantity = useProductStore((state) => state.updateQuantity);
  const [draft, setDraft] = useState(String(product.quantity));

  return (
    <div className="flex items-center justify-between rounded-lg border border-zinc-200 bg-white px-5 py-4 shadow-sm dark:border-zinc-700 dark:bg-zinc-900">
      <div className="flex flex-col gap-1">
        <span className="font-medium text-zinc-900 dark:text-zinc-50">{product.name}</span>
        <span className="text-sm text-zinc-500 dark:text-zinc-400">{product.category}</span>
      </div>
      <div className="flex items-center gap-4">
        <span className="font-semibold text-zinc-800 dark:text-zinc-200">
          ${product.price.toFixed(2)}
        </span>
        <div className="flex items-center gap-2">
          <label htmlFor={`qty-${product.id}`} className="text-sm text-zinc-500 dark:text-zinc-400">
            Qty
          </label>
          <Input
            id={`qty-${product.id}`}
            type="number"
            min={0}
            value={draft}
            onChange={(e) => setDraft(e.target.value)}
            className="w-16"
          />
        </div>
        <Button
          variant="outline"
          size="sm"
          onClick={() => {
            const parsed = parseInt(draft, 10);
            if (!isNaN(parsed) && parsed >= 0) {
              updateQuantity(product.id, parsed);
            } else {
              setDraft(String(product.quantity)); // revert invalid input
            }
          }}
        >
          Save
        </Button>
        <Button
          variant="destructive"
          size="sm"
          onClick={() => deleteProduct(product.id)}
          aria-label={`Delete ${product.name}`}
        >
          Delete
        </Button>
      </div>
    </div>
  );
}
```

**Key concepts:**

- `draft` is local React state — the Input is a controlled component that only writes to the Zustand store when "Save" is clicked.
- This is an intentional UX pattern: the user can type freely without triggering a store update on every keystroke.
- Invalid/negative values are rejected and the input reverts to the last saved quantity.

### 8. `app/page.tsx`

```tsx
import ProductList from "../components/ProductList";

export default function Home() {
  return (
    <div className="flex flex-col flex-1 items-center justify-center bg-zinc-50 font-sans dark:bg-zinc-950">
      <main className="flex flex-1 w-full flex-col items-center py-16 px-6">
        <ProductList />
      </main>
    </div>
  );
}
```

### 9. `app/layout.tsx`

```tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono, Inter } from "next/font/google";
import "./globals.css";
import { cn } from "@/lib/utils";

const inter = Inter({ subsets: ["latin"], variable: "--font-sans" });
const geistSans = Geist({ variable: "--font-geist-sans", subsets: ["latin"] });
const geistMono = Geist_Mono({ variable: "--font-geist-mono", subsets: ["latin"] });

export const metadata: Metadata = {
  title: "7b",
  description: "Generated by create next app",
};

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html
      lang="en"
      className={cn("h-full", "antialiased", geistSans.variable, geistMono.variable, "font-sans", inter.variable)}
    >
      <body className="min-h-full flex flex-col">{children}</body>
    </html>
  );
}
```

---

## Running the app

```bash
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## How persistence works end-to-end

```
Page loads
  └─> Zustand store created with initialProducts + hydrated: false
  └─> persist middleware reads "products-store" key from localStorage
        ├─ Key exists  → merges saved products into store → calls onRehydrateStorage callback → hydrated: true
        └─ Key missing → keeps initialProducts              → calls onRehydrateStorage callback → hydrated: true

ProductList renders
  └─> reads hydrated from store
        ├─ false → renders <Spinner /> (no flash of stale content)
        └─ true  → renders the product list

User adds / deletes / saves quantity
  └─> Zustand action runs → state updates → persist middleware writes { products: [...] } to localStorage
```

---

## What is Hydration?

The word "hydration" is used in two related but distinct contexts in this project. Understanding the difference is essential.

### 1. React / Next.js Hydration

When Next.js serves a page, it first renders the component tree to plain HTML on the server and sends that HTML to the browser. The browser displays this HTML immediately (fast first paint). Then React's JavaScript bundle loads, and React "hydrates" the page — it attaches event listeners and takes over control of the DOM so the page becomes interactive.

**The problem:** During server-side rendering, `localStorage` does not exist (it is a browser-only API). So the Zustand store initialises with `initialProducts` on the server. But the user's browser might have different data saved in `localStorage` from a previous visit. When React hydrates, if the component immediately reads from `localStorage` and renders different content than what the server rendered, React throws a **hydration mismatch warning** and the UI can flash or jump.

### 2. Zustand / localStorage Hydration (`persist` middleware)

Zustand's `persist` middleware has its own concept of hydration: the process of **reading the saved JSON from `localStorage` and merging it back into the store**. This happens asynchronously, after the JavaScript bundle loads.

When the store is first created the state is `initialProducts` and `hydrated: false`. Once the middleware finishes reading from `localStorage` it calls the `onRehydrateStorage` callback and the state becomes `hydrated: true`.

The `hydrated` flag is used in `ProductList.tsx` as a gate:

```tsx
if (!hydrated) return <Spinner className="size-6" />;
```

This prevents the component from rendering stale seed data until the real localStorage data is loaded, completely eliminating the React hydration mismatch. The user sees a spinner instead of a blank screen while waiting.

**Summary:**

| Term | Who uses it | What it means |
|---|---|---|
| React hydration | React / Next.js | React attaches to server-rendered HTML to make it interactive |
| Store hydration | Zustand `persist` | Middleware reads localStorage and restores saved state into the store |

---

## Deep Dive: `useProductStore.ts` — Every Line Explained

```ts
"use client";
```
This is a Next.js App Router directive. It marks this module as a **Client Component**, meaning it only runs in the browser. This is required because the file uses `localStorage` (a browser API) and Zustand hooks (which rely on React context / hooks that need the browser).

---

```ts
import { create } from "zustand";
```
`create` is Zustand's factory function. You call it with a function that receives `set` and `get` and returns the initial state + actions. It returns a custom React hook (`useProductStore`).

---

```ts
import { createJSONStorage, persist } from "zustand/middleware";
```
- `persist` — a Zustand middleware that wraps your store creator. It intercepts every `set` call and serialises the state to a storage backend (here, `localStorage`). On startup it reads back the saved state and merges it in.
- `createJSONStorage` — a helper that adapts any Web Storage API object (`localStorage`, `sessionStorage`) into the interface Zustand's `persist` middleware expects. It handles `JSON.stringify` on write and `JSON.parse` on read.

---

```ts
export type Product = {
  id: number;
  name: string;
  price: number;
  category: string;
  quantity: number;
};
```
A TypeScript type describing the shape of a single product. Exporting it allows `ProductItem.tsx` to use `Product` as a prop type without duplicating the definition.

---

```ts
type ProductStore = {
  products: Product[];
  hydrated: boolean;
  setHydrated: (value: boolean) => void;
  addProduct: () => void;
  deleteProduct: (id: number) => void;
  updateQuantity: (id: number, quantity: number) => void;
};
```
The TypeScript type for the entire store — both **state** (`products`, `hydrated`) and **actions** (`setHydrated`, `addProduct`, `deleteProduct`, `updateQuantity`). Zustand treats state and actions uniformly inside one object.

- `hydrated: boolean` — tracks whether the `persist` middleware has finished reading from localStorage. Starts `false`, becomes `true` after rehydration.
- `setHydrated` — a setter action so the `onRehydrateStorage` callback can flip `hydrated` to `true`.

---

```ts
const initialProducts: Product[] = [ ... ];
```
The hard-coded seed data used when the app runs for the first time (no localStorage entry yet). On subsequent page loads, Zustand replaces this with whatever was saved in localStorage.

---

```ts
export const useProductStore = create<ProductStore>()(
```
- `create<ProductStore>()` — the double-call `()()` is needed because `persist` is a middleware (curried function). The outer `()` receives the generic type; the inner `()` receives the store creator wrapped in `persist`.
- The result is exported as `useProductStore`, a React hook that components call to subscribe to slices of the store.

---

```ts
  persist(
    (set, get) => ({ ... }),
    { /* options */ }
  )
```
`persist` takes two arguments:
1. **The store creator** — the same `(set, get) => ({...})` function you'd pass to `create` normally.
2. **Options object** — configures storage key, backend, what to save, and lifecycle callbacks.

---

```ts
      products: initialProducts,
```
Initialises `products` with the seed data. `persist` will overwrite this with localStorage data if it exists.

---

```ts
      hydrated: false,
```
Starts as `false`. This means "localStorage has not been read yet". Components that depend on persisted data should wait for this to become `true` before rendering.

---

```ts
      setHydrated: (value) => set({ hydrated: value }),
```
A simple setter action. Calling `set({ hydrated: true })` merges `{ hydrated: true }` into the current state (Zustand's `set` does a shallow merge by default). This is called by `onRehydrateStorage` once hydration completes.

---

```ts
      addProduct: () => {
        const products = get().products;
```
`get()` returns the current state snapshot **outside of React's render cycle**. This is how actions read current state without subscribing to it. Here it reads the current array to calculate the next id.

---

```ts
        const nextId = products.length
          ? Math.max(...products.map((p) => p.id)) + 1
          : 1;
```
Finds the highest existing `id` and adds 1. Using `Math.max` with spread ensures that even if products are deleted (leaving gaps), the new id is always unique and greater than all existing ids. The ternary handles the empty-array edge case (spreading an empty array into `Math.max` returns `-Infinity`).

---

```ts
        const nextIndex = products.length + 1;
```
Used only for the display name (`"New Product 3"`). This is the human-readable count, not the id.

---

```ts
        set({
          products: [ ...products, { id: nextId, name: `New Product ${nextIndex}`, ... } ],
        });
```
`set` in Zustand does a **shallow merge** — you only need to provide the keys you want to change. Spreading `...products` and appending the new item creates a new array reference, which triggers a re-render in all subscribed components.

---

```ts
      deleteProduct: (id) =>
        set((state) => ({
          products: state.products.filter((p) => p.id !== id),
        })),
```
The callback form of `set` — `set(fn)` — receives the current state and returns a partial update. This form is preferred when the new state depends on the previous state (avoids stale closure issues). `filter` returns a new array without the deleted item.

---

```ts
      updateQuantity: (id, quantity) =>
        set((state) => ({
          products: state.products.map((p) =>
            p.id === id ? { ...p, quantity } : p
          ),
        })),
```
`map` iterates every product. For the matching `id`, it spreads the existing product and overrides only `quantity` (shorthand property: `{ quantity }` equals `{ quantity: quantity }`). Every other product is returned unchanged. This immutable update pattern ensures React detects the state change.

---

```ts
    {
      name: "products-store",
```
The **key** used to store data in `localStorage`. Open DevTools → Application → Local Storage and you'll see an entry named `products-store` whose value is the JSON-serialised state.

---

```ts
      storage: createJSONStorage(() => localStorage),
```
Tells `persist` to use `localStorage` as the backend. The factory function `() => localStorage` is used instead of passing `localStorage` directly because on the server `localStorage` is `undefined` — the factory is only evaluated in the browser.

---

```ts
      partialize: (state) => ({ products: state.products }),
```
`partialize` is a function that **selects which parts of the state get saved** to storage. Here only `{ products }` is persisted. `hydrated` and `setHydrated` are intentionally excluded because:
- They are runtime-only flags — there is no value in saving them.
- If `hydrated: true` were saved, on next load it would be restored before the middleware actually finished reading, breaking the gate logic.

Without `partialize`, Zustand would save the entire store (including `hydrated` and action functions), which is wasteful and can cause errors when deserialising functions from JSON.

---

```ts
      onRehydrateStorage: () => (state) => {
        state?.setHydrated(true);
      },
```
`onRehydrateStorage` is a lifecycle hook. It returns a **callback** that Zustand calls once after the stored data has been merged into the store. The double-arrow pattern means:
- The outer function `() =>` is called by Zustand when the store is created (before storage is read).
- The inner function `(state) =>` is the callback Zustand calls **after** rehydration finishes, passing the now-hydrated state.

`state?.setHydrated(true)` uses optional chaining because `state` can be `undefined` if an error occurred during storage parsing. This sets `hydrated: true`, unblocking any components that returned `null` while waiting.

---

## Getting Started

```bash
# install dependencies
pnpm install

# run dev server
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.
