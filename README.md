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
| shadcn/ui | Pre-built UI components (Button, Input) |
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
```

This creates `components/ui/button.tsx` and `components/ui/input.tsx`.

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

  if (!hydrated) return null; // wait for localStorage to be read before rendering

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
- `if (!hydrated) return null` prevents the component from rendering before localStorage has been read; without this, Next.js may show a hydration mismatch warning.

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
        ├─ false → returns null (no flash of stale content)
        └─ true  → renders the product list

User adds / deletes / saves quantity
  └─> Zustand action runs → state updates → persist middleware writes { products: [...] } to localStorage
```

---

## Getting Started

```bash
# install dependencies
pnpm install

# run dev server
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.
