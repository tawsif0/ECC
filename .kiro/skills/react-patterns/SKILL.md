---
name: react-patterns
description: React 18/19 patterns including hooks discipline, server/client component boundaries, Suspense + error boundaries, form actions, data fetching, state management decision trees, and accessibility-first composition. Use when writing or reviewing React components.
origin: ECC
---

# React Patterns

Idiomatic React 18/19 patterns for building robust, accessible, performant component trees.

## When to Activate

- Writing or modifying React function components, custom hooks, or component trees
- Reviewing JSX/TSX files
- Designing state shape or component composition
- Migrating class components or older `forwardRef`/`useEffect`-heavy code
- Choosing between local state, lifted state, context, and external stores
- Working with Server Components / Client Components (Next.js App Router, RSC)
- Implementing forms with React 19 actions or controlled inputs
- Wiring data fetching with TanStack Query / SWR / RSC

## Core Principles

### 1. Render is a Pure Function of Props and State

```tsx
// Good: derive during render
function Cart({ items }: { items: CartItem[] }) {
  const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  return <span>{formatMoney(total)}</span>;
}

// Bad: derived state stored separately
function Cart({ items }: { items: CartItem[] }) {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((sum, i) => sum + i.price * i.qty, 0));
  }, [items]);
  return <span>{formatMoney(total)}</span>;
}
```

Derived state in `useEffect` adds a render cycle, can desync, and obscures the data flow.

### 2. Side Effects Outside Render

Effects, mutations, network calls, and subscriptions live in event handlers or `useEffect` — never in the render body.

### 3. Composition Over Inheritance

React has no inheritance model for components. Compose with `children`, render props, or component props.

## Hooks Discipline

See [rules/react/hooks.md](../../rules/react/hooks.md) for the full ruleset. Highlights:

- Top-level only, never conditional
- Cleanup every subscription, interval, listener
- Functional updater (`setX(prev => prev + 1)`) when new state depends on old
- Default position: do not memoize — add `useMemo`/`useCallback` only when a profiler or a dependency chain proves it matters
- Extract a custom hook only when the same hook sequence appears in 2+ components

## State Location Decision Tree

```
Used by one component?
  -> useState inside it

Used by parent + a few descendants?
  -> lift to nearest common ancestor

Used across distant branches AND low-frequency reads (theme, auth, locale)?
  -> React Context

High-frequency updates shared across the tree?
  -> external store (Zustand, Jotai, Redux Toolkit)

Derived from a server?
  -> server-state library (TanStack Query, SWR, RSC fetch)
```

Most pages do not need context or a global store. Resist abstraction until duplicated lifting becomes painful.

## Server / Client Components (RSC)

```tsx
// Server Component - default, async, never ships JS for itself
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  if (!product) notFound();
  return <ProductView product={product} />;
}

// Client Component - opt in with "use client"
"use client";
export function AddToCartButton({ productId }: { productId: string }) {
  const [pending, startTransition] = useTransition();
  return (
    <button
      disabled={pending}
      onClick={() => startTransition(() => addToCart(productId))}
    >
      {pending ? "Adding..." : "Add to cart"}
    </button>
  );
}
```

Boundaries:

- Server -> Client: pass serializable props or `children`
- Client -> Server: invoke Server Actions via `<form action={...}>` or imperatively from event handlers
- Never `import` a Server Component from a Client Component file — compose them via `children` instead

## Suspense + Error Boundaries

```tsx
<ErrorBoundary fallback={<ErrorView />}>
  <Suspense fallback={<UserSkeleton />}>
    <UserDetail id={id} />
  </Suspense>
</ErrorBoundary>
```

- Place Suspense boundaries close to the data, not at the route root — progressively reveal content
- Error Boundary remains a class API; use `react-error-boundary` for a hook-friendly wrapper
- A boundary catches errors thrown during render, lifecycle, and constructors of its children — NOT in event handlers or async code

## Forms

### React 19 form actions (preferred for new code)

> **React 19+**: This example uses `useActionState`. For React 18, use `useFormState` from `react-dom` instead.

```tsx
"use client";
import { useActionState } from "react";

const initial = { error: null as string | null };

async function updateUserAction(_prev: typeof initial, formData: FormData) {
  "use server";
  const parsed = UserSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: "Invalid input" };
  await db.user.update({ where: { id: parsed.data.id }, data: parsed.data });
  return { error: null };
}

export function UserForm() {
  const [state, formAction, pending] = useActionState(updateUserAction, initial);
  return (
    <form action={formAction}>
      <input name="name" required />
      <button type="submit" disabled={pending}>Save</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

### Controlled inputs

Use controlled when the value drives other UI, formats on every keystroke, or implements real-time validation.

### Complex forms

For multi-step forms, dynamic field arrays, or cross-field validation: use a library (React Hook Form, TanStack Form). Roll-your-own state management for forms past trivial complexity is a maintenance trap.

## Data Fetching Decision Matrix

| Need | Tool |
|---|---|
| Per-request data in Next.js App Router | RSC `await fetch()` |
| Client-side cache + mutations + invalidation | TanStack Query |
| Lightweight client cache + revalidation | SWR |
| Real-time subscriptions | Server-Sent Events, WebSockets, or the lib's subscription API |
| One-off fire-and-forget | `fetch()` in an event handler |

Avoid `useEffect` + `fetch` for application data — race conditions, no cache, no retry, no Suspense integration.

## Composition Recipes

### Slot via `children`

```tsx
<Layout>
  <Header />
  <Main>{content}</Main>
</Layout>
```

### Named slots

```tsx
<Page header={<Nav />} sidebar={<Filters />}>
  <Results />
</Page>
```

### Compound components (shared state via Context)

```tsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="profile"><Profile /></Tabs.Panel>
  <Tabs.Panel value="settings"><Settings /></Tabs.Panel>
</Tabs>
```

### Render prop / function-as-child

Useful when the parent needs to pass parameters to the rendered output:

```tsx
<DataLoader id={id}>
  {({ data, isLoading }) => isLoading ? <Spinner /> : <UserCard user={data} />}
</DataLoader>
```

Modern alternative: a hook (`useData(id)`) returning the same shape — usually cleaner.

## Performance

### When `React.memo` Actually Helps

Wrap a component in `React.memo` only when:

1. It re-renders frequently
2. Its props are usually the same between renders
3. Its render is measurably expensive

`React.memo` adds an equality check on every render. If props differ on most renders, the check is pure overhead.

### Avoiding Render Cascades

- Lift state down rather than up where possible
- Split context: one context per concern, so a change to `themeContext` does not re-render auth consumers
- Use `useSyncExternalStore` for external state libraries — required for safe concurrent rendering

### Lists

- Provide stable `key` props (database id, not array index)
- Virtualize long lists with `@tanstack/react-virtual` or `react-window` once visible item count exceeds ~50 with non-trivial rows

## Accessibility-First Composition

- Always render semantic HTML (`<button>`, `<a>`, `<nav>`, `<main>`) before reaching for `role` attributes
- Every interactive element must be reachable by keyboard
- Form inputs need labels — `<label htmlFor>` or `aria-label` if visually labeled by an icon
- Manage focus on route changes and modal open/close
- Run `axe` in component tests (see [skills/react-testing](../react-testing/SKILL.md))
- Cross-link: [skills/accessibility/SKILL.md](../accessibility/SKILL.md) covers WCAG criteria and pattern libraries

## Routing

This skill is router-agnostic. The patterns above work with React Router, TanStack Router, Next.js App Router, Remix Router. Router-specific patterns (loaders, actions, nested layouts) follow the router's documentation — those are framework concerns layered on top of React core.

## Out of Scope (Pointer Sections)

- **Next.js specifics**: App Router data loading, Route Handlers, Middleware, Parallel Routes — separate concern, use Next.js docs
- **React Native**: Platform-specific patterns differ enough to warrant a separate `react-native-patterns` skill (not present yet)
- **Remix**: Loader/action conventions overlap with RSC but follow Remix docs

## Related

- Rules: [rules/react/](../../rules/react/) — coding-style, hooks, patterns, security, testing
- Skills: [react-performance](../react-performance/SKILL.md) for the Vercel-derived performance ruleset, [frontend-patterns](../frontend-patterns/SKILL.md) for cross-framework UI concerns, [accessibility](../accessibility/SKILL.md), [angular-developer](../angular-developer/SKILL.md) for framework comparison
- Agents: `react-reviewer` for code review, `react-build-resolver` for build/bundler errors
- Commands: `/react-review`, `/react-build`, `/react-test`

## Examples

### Custom hook for debounced search

```tsx
function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

function SearchBox() {
  const [query, setQuery] = useState("");
  const debounced = useDebounce(query, 300);
  const { data } = useQuery({
    queryKey: ["search", debounced],
    queryFn: () => searchApi(debounced),
    enabled: debounced.length > 0,
  });
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results items={data ?? []} />
    </>
  );
}
```

### Optimistic UI with React 19 `useOptimistic`

```tsx
"use client";
import { useOptimistic } from "react";

export function MessageList({ messages }: { messages: Message[] }) {
  const [optimistic, addOptimistic] = useOptimistic(
    messages,
    (state, newMessage: Message) => [...state, newMessage],
  );

  async function send(formData: FormData) {
    const text = String(formData.get("text"));
    addOptimistic({ id: "pending", text, sender: "me" });
    await saveMessage(text);
  }

  return (
    <>
      <ul>{optimistic.map((m) => <li key={m.id}>{m.text}</li>)}</ul>
      <form action={send}>
        <input name="text" />
        <button type="submit">Send</button>
      </form>
    </>
  );
}
```

### Splitting context to avoid render cascades

```tsx
// Two contexts: one rarely changes, one frequently
const ThemeContext = createContext<Theme>("light");
const NotificationsContext = createContext<Notification[]>([]);

// A component that only consumes ThemeContext does NOT re-render when notifications change
```

## Hand-rolled async UI: four traps a query library would have handled

When you can't reach for TanStack Query (a one-off widget, a legacy screen, a plain `fetch` in a
component), you inherit the problems it solves. Each of these shipped as a real bug.

### 1. Out-of-order responses — guard with a request sequence, not just a debounce

A debounce reduces requests; it does not order them. A slow older query resolving after a newer one
overwrites correct results with stale ones (typing `abandon` can leave you looking at results for `ab`).
Stamp each request and discard anything that isn't the latest:

```tsx
const reqId = useRef(0);
async function search(q: string) {
  const id = ++reqId.current;
  const data = await searchApi(q);
  if (id !== reqId.current) return;      // a newer request already started — drop this response
  setResults(data);
}
// Clearing the input / toggling direction must ALSO bump reqId.current and cancel the pending debounce,
// or an in-flight response repopulates a box the user just emptied.
```
Use the same guard on admin list views (`listReqId`) — filter and tab switches race identically.

### 2. Session expiry belongs in one place

A token that expires mid-session makes every subsequent write fail with a generic toast, leaving the user
stuck with no way out. Route every response through one handler: on **401/403**, clear the stored session
and redirect to login with a "Session expired" message, instead of surfacing a raw error at each call
site. Put it in the fetch wrapper/interceptor so call site #147 can't forget.

### 3. Canvas → image export (share / download cards)

- **`canvas.toBlob` can resolve `null`** (tainted canvas, OOM). Using the blob unconditionally emits a
  blank or corrupt PNG. Null-check it, and share one `renderCardBlob()` helper between Share and Download
  so the check exists exactly once.
- **A failed image load taints the canvas.** If the logo/avatar loader falls back to resolving the *raw
  URL* on error, that cross-origin image taints the `html2canvas` capture and the export dies. Resolve
  `null` on error and render the card without the image.
- **Concurrent Share + Download need a synchronous lock.** React state lags a render, so an
  `isGenerating` *state* check lets a double-tap through. Use a ref:
  `if (generatingRef.current) return; generatingRef.current = true;` in a `try/finally`.
- **A tight `line-height` inside `overflow: hidden` clips serif ascenders in the rasterized output.** The
  card looks right in the browser (glyphs overflow their line box harmlessly) but the capture crops them,
  so a display serif's `f`/`h`/`l`/diacritics lose their tops in the exported PNG only. Keep the card's
  headline at `lineHeight ≥ 1.25` rather than a designer's `1.05`, or drop the `overflow: hidden` on that
  element. Always review the *exported file*, not the on-screen card — they are two different renderers.

### 4. Defensive defaults on API-shaped state

A malformed or partial response leaves `pagination.total` undefined and crashes the results-count render.
Merge over defaults at the boundary — `setPagination({ total: 0, hasMore: false, ...res.pagination })` —
rather than optional-chaining at every read site.

> Interrupting a flow with a login? The pending-intent-with-TTL handoff lives in `frontend-patterns`
> ("Resume-an-action-after-login") — don't re-implement it here.

## Deep links to records that may not exist

A link or chip pointing at a record that was never created (a synonym absent from the dictionary, a
deleted job, a bad `?id=` param) must not be a silent dead click. Either don't render it as a link, or
show a transient "…isn't available yet" notice. Pre-resolve which related ids actually exist and render
only those as links — the check is one `find({ key: { $in: [...] } })`, not N requests.

**Resolve a deep link with an exact-match endpoint — never a fuzzy search plus a client-side equality
check.** The tempting shortcut for `?word=good` / `?ref=ACME` is to reuse the list search
(`GET /items?q=good&limit=5`) and open whichever result matches exactly. It fails precisely for the most
*common* values: a substring/regex search spans several fields, so dozens of other records mention the
term (as a synonym, a tag, a description) and rank ahead of it — the exact record never appears in the
first N, the equality check misses, and the deep link **silently does nothing** with no error to explain
it. Rarer values work fine, which is why it survives testing. Add `GET /items/by-key/:key` doing a
`findOne({ keyLower })` and point the boot effect at that; keep the fuzzy search for the search box only.

**Highlighting a term inside a sentence: `\b` is ASCII-only.** `new RegExp(\`\\b${esc(term)}\\b\`, 'gi')`
silently fails to match whenever the term or its neighbours are non-Latin (Bangla, Cyrillic, CJK,
accented Latin), so bilingual example sentences render unhighlighted. Split on a plain capture group and
compare case-insensitively per part instead — the equality check, not the regex, is what guarantees
correctness:
```js
const parts = sentence.split(new RegExp(`(${escapeRegExp(term)})`, 'gi'));
parts.map(p => p.toLowerCase() === term.toLowerCase() ? <mark key={p}>{p}</mark> : p);
```

## Module-level media objects keep streaming after the component is gone

A shared `const audio = new Audio()` (or a single `<video>`/`AudioContext`/`MediaRecorder`) parked at module
scope is convenient — one instance, no re-creation, easy "stop the previous clip before starting the next".
But nothing in a component's unmount touches it, so navigating away mid-playback in an SPA leaves the
request **open and streaming**; a proxied/generated source (a TTS endpoint, an on-demand transcode) then
holds a live server connection per navigation, and they accumulate for the session. `src=""` alone isn't
enough on some browsers — pause, clear, then load:
```js
// media.js
let el = null;
export function play(src){ stop(); el = new Audio(src); return el.play(); }
export function stop(){ if (!el) return; el.pause(); el.src = ''; el.load(); el = null; }
// every component that can start playback:
useEffect(() => stop, []);      // cleanup on unmount, and on route change via the same unmount
```
The module owning the element must **export the stopper** — a singleton with no public `stop()` is the bug.
