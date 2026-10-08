---
name: react-native-patterns
description: React Native and Expo app patterns — Expo Router navigation, state separation (server/client/route/form), TanStack Query data fetching with Zod, performant lists, NativeWind/StyleSheet styling, native APIs, and secure storage. Use when building or editing React Native / Expo screens, components, navigation, or data layers.
origin: ECC
---

# React Native / Expo Patterns

Practical patterns for building production React Native apps with Expo. Covers navigation, state, data fetching, lists, styling, and native APIs. Pairs with the `rules/react-native/` ruleset: rules say *what* to enforce, this skill shows *how*.

Libraries named below (NativeWind, Zustand/Jotai, TanStack Query) are common, well-established options shown for illustration — the patterns matter more than the specific package, and any equivalent works. Zod is used for validation to stay consistent with ECC's existing `typescript/` rules.

These patterns assume the managed Expo workflow (Expo Router, EAS, `expo-*` modules) on the New Architecture (the default in recent Expo SDKs, mandatory from SDK 55+). They do NOT assume the browser DOM — React Native has no `<div>`, no URL bar, and no web data-fetching defaults.

## When to Activate

Use this skill when:

- Building or editing React Native / Expo screens, components, or navigation
- Setting up routing with Expo Router (file-based `app/` directory)
- Deciding where state belongs (server cache vs client store vs route params vs form)
- Wiring data fetching with TanStack Query and validating responses with Zod
- Rendering long or heavy lists
- Choosing or applying a styling approach (NativeWind or StyleSheet)
- Accessing native device APIs (camera, location, notifications) or secure storage
- Reviewing RN code for mobile-specific issues

Do NOT use the web/React-DOM patterns here — URL-as-state, `<div>`, and SWR-for-browser do not apply to React Native.

## Core Concepts

### Project structure (Expo Router)

File-based routing under `app/`. Keep route files thin: they read and validate params, then delegate to a screen component that lives in `components/` or `features/`.

```
app/
  _layout.tsx          # root stack
  (tabs)/
    _layout.tsx        # tab navigator
    index.tsx          # Home
  user/[id].tsx        # dynamic route
components/
features/
  user/UserProfile.tsx
```

### Navigation: validate route params

Deep links and dynamic routes deliver untrusted strings. Validate them with Zod before use.

```tsx
// app/user/[id].tsx
import { useLocalSearchParams, router } from 'expo-router'
import { z } from 'zod'
import { UserProfile } from '@/features/user/UserProfile'

const Params = z.object({ id: z.string().uuid() })

export default function UserRoute() {
  const parsed = Params.safeParse(useLocalSearchParams())
  if (!parsed.success) {
    router.replace('/not-found')
    return null
  }
  return <UserProfile userId={parsed.data.id} />
}
```

### State: keep concerns separate

Do not duplicate server data into a client store. Each concern has its own home.

| Concern | Common choices |
|---------|------|
| Server state (remote data) | a server-cache library (TanStack Query, SWR) |
| Client/UI state | a lightweight store (Zustand, Jotai) or Context |
| Route/navigation state | Expo Router params |
| Form state | a form library (e.g. React Hook Form) + schema validation |
| Secrets / tokens | `expo-secure-store` |
| Non-secret persistence | `AsyncStorage` / MMKV |

Prefer local `useState` until state genuinely needs sharing.

### Data fetching: a cache library + Zod

Use a server-cache library (TanStack Query, SWR) instead of fetch-in-`useEffect`. Validate at the boundary and infer types from the schema. Handle loading, error, and empty states explicitly. (Example uses TanStack Query.)

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { z } from 'zod'

const User = z.object({ id: z.string(), email: z.string().email() })
type User = z.infer<typeof User>

export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: async (): Promise<User> => User.parse(await api.getUser(id)),
  })
}

export function useUpdateEmail(id: string) {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (email: string) => api.updateEmail(id, email),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['user', id] }),
  })
}
```

### Lists: virtualize, never map a big array in a ScrollView

```tsx
import { FlatList } from 'react-native'

<FlatList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={renderItem}          // memoized
  initialNumToRender={10}
  windowSize={5}
/>
```

Use `FlashList` (Shopify) for large or heterogeneous lists.

### Styling: pick one system

`StyleSheet.create()` is the framework-native option; utility-class libraries (e.g. NativeWind) are a common alternative. Choose one and stay consistent. Never build style objects inline in JSX on hot paths.

```tsx
// NativeWind
<View className="p-4 rounded-2xl bg-white">
  <Text className="text-base font-semibold">Hello</Text>
</View>

// StyleSheet
const styles = StyleSheet.create({ card: { padding: 16, borderRadius: 16, backgroundColor: '#fff' } })
<View style={styles.card}>...</View>
```

### Native APIs: wrap in hooks, clean up effects

Keep Expo SDK calls and subscriptions inside `use*` hooks, not in JSX. Always clean up.

```tsx
import { useEffect, useState } from 'react'
import * as Location from 'expo-location'

type LocationState =
  | { status: 'loading' }
  | { status: 'denied' }
  | { status: 'granted'; coords: Location.LocationObjectCoords }

export function useCurrentLocation() {
  // Track status, not just coords — so the UI can tell "still loading" apart
  // from "permission denied" and show an actionable message.
  const [state, setState] = useState<LocationState>({ status: 'loading' })

  useEffect(() => {
    let active = true
    ;(async () => {
      const { status } = await Location.requestForegroundPermissionsAsync()
      if (status !== 'granted') {
        if (active) setState({ status: 'denied' })
        return
      }
      const pos = await Location.getCurrentPositionAsync({})
      if (active) setState({ status: 'granted', coords: pos.coords })
    })()
    return () => { active = false }   // ignore stale result after unmount
  }, [])

  return state
}
```

### Secure storage for tokens

```tsx
import * as SecureStore from 'expo-secure-store'

await SecureStore.setItemAsync('auth_token', token)   // Keychain / Keystore
const token = await SecureStore.getItemAsync('auth_token')
```

### Retrofitting an auth header onto an app with no API client

When screens already call bare `fetch()` from dozens of call sites, wrapping `globalThis.fetch` once at
app boot beats refactoring them all into a client. Do this in the root component, before first render:

```tsx
const realFetch = globalThis.fetch
globalThis.fetch = (input, init = {}) => {
  const url = typeof input === 'string' ? input : input.url
  // ONLY your own backend — never leak the token to third-party hosts
  if (!url.startsWith(BACKEND_URL)) return realFetch(input, init)
  return realFetch(input, {
    ...init,
    headers: { ...init.headers, 'x-auth-token': getToken() },
  })
}
```

- **The `startsWith(BACKEND_URL)` check is the whole safety property.** An unconditional patch attaches your
  session token to every image CDN, analytics ping, and OAuth callback the app touches.
- Capture `realFetch` **once**, at module scope. Patching twice (Fast Refresh, a re-run `useEffect`) makes the
  wrapper call itself — infinite recursion that only shows up on a reloaded dev build.
- Read the token **lazily inside the wrapper** (`getToken()`), not captured at patch time, or the app keeps
  sending the pre-login `undefined` until the next cold start.
- This is a **migration tool, not an architecture.** It's invisible to anyone reading a call site, so it can't
  be typed, mocked, or per-request-overridden. Use it to ship auth across a large legacy surface today, then
  collapse call sites onto a real client (or TanStack Query's `queryFn`) as you touch them.

## Code Examples

### A full screen: route → query → list → states

```tsx
// app/(tabs)/orders.tsx
import { memo, useCallback } from 'react'
import { FlatList, Text, View } from 'react-native'
import { useQuery } from '@tanstack/react-query'
import { z } from 'zod'

const OrderSchema = z.object({ id: z.string(), total: z.number(), status: z.string() })
const OrdersSchema = z.array(OrderSchema)
type Order = z.infer<typeof OrderSchema>

function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: async () => OrdersSchema.parse(await api.listOrders()),
  })
}

// Memoized so its reference is stable across renders (see the lists guidance).
const OrderRow = memo(function OrderRow({ item }: { item: Order }) {
  return (
    <View className="px-4 py-3 border-b border-neutral-200">
      <Text className="font-medium">#{item.id}</Text>
      <Text className="text-neutral-500">{item.status} · ${item.total}</Text>
    </View>
  )
})

export default function OrdersScreen() {
  const { data, isLoading, isError, refetch, isRefetching } = useOrders()
  const renderItem = useCallback(({ item }: { item: Order }) => <OrderRow item={item} />, [])

  if (isLoading) return <Centered><Text>Loading…</Text></Centered>
  if (isError) return <Centered><Text accessibilityRole="alert">Could not load orders.</Text></Centered>
  if (!data?.length) return <Centered><Text>No orders yet.</Text></Centered>

  return (
    <FlatList
      data={data}
      keyExtractor={(o) => o.id}
      onRefresh={refetch}
      refreshing={isRefetching}
      renderItem={renderItem}
    />
  )
}
```

### A form: React Hook Form + Zod resolver

```tsx
import { useForm, Controller } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { TextInput, Button, Text } from 'react-native'

const Schema = z.object({ email: z.string().email('Invalid email') })
type FormValues = z.infer<typeof Schema>

export function EmailForm({ onSubmit }: { onSubmit: (v: FormValues) => void }) {
  const { control, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(Schema),
    defaultValues: { email: '' },
  })

  return (
    <>
      <Controller
        control={control}
        name="email"
        render={({ field: { value, onChange, onBlur } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            autoCapitalize="none"
            keyboardType="email-address"
            accessibilityLabel="Email address"
          />
        )}
      />
      {errors.email && <Text accessibilityRole="alert">{errors.email.message}</Text>}
      <Button title="Save" onPress={handleSubmit(onSubmit)} />
    </>
  )
}
```

## Anti-Patterns

```tsx
// WRONG: large array mapped inside a ScrollView (no virtualization, janky, high memory)
<ScrollView>{items.map((i) => <Row key={i.id} item={i} />)}</ScrollView>
// RIGHT: FlatList / FlashList

// WRONG: server data copied into a client store (two sources of truth, stale data)
const useStore = create((set) => ({ users: [], setUsers: (u) => set({ users: u }) }))
useEffect(() => { getUsers().then(setUsers) }, [])
// RIGHT: useQuery owns server state; derive what you need

// WRONG: tokens in AsyncStorage (not encrypted)
await AsyncStorage.setItem('auth_token', token)
// RIGHT: expo-secure-store

// WRONG: trusting deep-link params
const { id } = useLocalSearchParams(); fetchUser(id)
// RIGHT: validate with Zod before use

// WRONG: inline style object recreated every render on a hot path
<View style={{ padding: 16, backgroundColor: '#fff' }} />
// RIGHT: StyleSheet.create at module scope, or NativeWind className

// WRONG: real secret shipped in the bundle
const STRIPE_SECRET = 'sk_live_...'
// RIGHT: keep privileged calls server-side; ship only public keys protected by backend rules
```

## Best Practices

- Keep route files thin; put logic in screen components and `use*` hooks.
- Validate every external input (API responses, route params, push payloads) with Zod.
- Let TanStack Query own server state; keep client stores small.
- Always render loading, error, and empty states — never just a spinner with no fallback.
- Virtualize lists; memoize `renderItem`; provide a stable `keyExtractor`.
- Use `react-native-reanimated` for animation (UI thread); avoid heavy work on the JS thread.
- Store tokens in `expo-secure-store`; never trust the client for authorization.
- Respect safe areas, Dynamic Type, and accessibility roles/labels from the start.
- Confirm New Architecture compatibility for every native dependency before release.

## Zero-asset visual polish + native push (from a production social app)

Techniques that make an RN UI feel premium/gamified using only `react-native-svg` — no image assets, no `expo-linear-gradient`.

- **Gradient fill via SVG (the load-bearing trick).** Absolutely-position an `<Svg>` behind content, define a `<LinearGradient>` in `<Defs>` with a **per-variant unique `id`** (e.g. `gb_${variant}`) so multiple instances don't collide, fill a `<Rect width="100%" height="100%">`. Wrap the button in `rounded-2xl overflow-hidden` and give it a brand-tinted shadow (`shadowColor` = the gradient's start color). Reusable CTA gradient pairs `[from,to]`: purple `['#a855f7','#6366f1']`, pink `['#f43f5e','#a855f7']`, blue `['#3b82f6','#06b6d4']`, green `['#10b981','#06b6d4']`, gold `['#f59e0b','#f43f5e']`, cyan `['#06b6d4','#3b82f6']`, danger `['#ef4444','#f43f5e']`.
- **Faux-3D glossy icons** — make a flat SVG read as a shaded sphere with 4 stacked layers: (1) drop shadow `<Ellipse cy=42 rx=14 ry=3.4 fill=#000 opacity=0.18/>`; (2) depth/rim circle offset down (`cy=26`) with a vertical `LinearGradient`; (3) face circle (`cy=23`) with an **off-center `RadialGradient`** (`cx=38% cy=32%`, light→mid→dark stops) = spherical shading; (4) glossy highlight `<Ellipse cx=18.5 cy=16 rx=7 ry=3.6 fill=#FFF opacity=0.45/>`.
- **Gamified level/tier ring** — a reusable avatar frame. Tier table `{min,name,c:[grad],ring,glow,gems,crown}`; `tierForLevel` reverse-finds the highest tier where `level >= min`. Render: radial outer glow, gradient stroke ring, gems placed on a circle via trig (`a = 2π·i/n − π/2; x=cx+r·cos(a)`), conditional crown `Path` for top tiers, avatar clipped in a circle with a letter fallback, absolute level-badge pill. Ladder: Bronze/Silver/Gold/Platinum/Diamond/Legendary at levels 0/10/20/40/60/80.
- **Pulsing attention FAB** — loop an `Animated.Value 0→1`; render 2 absolute glow rings mapping the value to `opacity [0.5,0]` and `scale [1, 1.6/2.1]` (`useNativeDriver:true`); reset with a `duration:0` timing back to 0.
- **Two-choice toggle-card selector** — a pair of large tappable cards; active state swaps `backgroundColor`+`borderColor` to the option's brand color and inverts icon/text to white; disabled CTA greys to `#cfc6ea`. Standard onboarding pattern (gender/plan/role pick).
- **FCM push via `@react-native-firebase/messaging` (modular API).** Lazily `require` the native module inside a `try/catch` so a build without Firebase no-ops instead of crashing: `function fb(){ try { return {app:getApp(), m:require('@react-native-firebase/messaging')} } catch { return null } }`. Android 13+ gate: `if (Platform.OS==='android' && Number(Platform.Version) >= 33) await PermissionsAndroid.request(PERMISSIONS.POST_NOTIFICATIONS)`. Flow: `requestPermission → getToken → POST {email,token,platform}` to the backend, all fire-and-forget (`.catch(()=>{})`). Return the `onMessage` unsubscribe fn directly from the effect for clean teardown. **Anti-pattern to avoid:** hard-coding `const BACKEND_URL = __DEV__ ? '…' : '…'` in every file (both branches identical) — use one shared config/env constant instead.

## More native-integration patterns
- **Native-SDK singleton with a promise queue.** For a heavy native engine (video/RTC/audio SDK) that
  must never be created or destroyed twice concurrently, hold one module-level instance and serialize
  every lifecycle op through a chained promise: `queue = queue.then(() => doCreateOrDestroy())`. Only
  recreate when a config key (e.g. the app/room id) actually changes; overlapping `join`/`leave` calls
  then can't race a half-built engine. Keep the SDK's **keys in env / remote config**, never baked into
  the source (a hard-coded `appID`/`appSign` ships to every device and can't be rotated).
- **State-machine navigation (no nav library).** For a small app you can drive screens with a single
  `activeTab`/`screen` `useState` in the root and switch on it, plus an Android hardware-back handler:
  `useEffect(() => { const sub = BackHandler.addEventListener('hardwareBackPress', () => { if (screen
  !== 'home') { setScreen('home'); return true; } return false; }); return () => sub.remove(); })`.
  Return `true` to consume the back press, `false` to let the OS exit. Fine until deep-linking or
  many stacks push you to a real router.
- **Dependency-free date-range picker (skip the native date-picker module).** Adding a native picker for
  one filter costs an autolink, a platform-behaviour split (Android dialog vs iOS spinner) and a release
  risk. A `Modal` with three `ScrollView` columns — Year / Month / Day — is ~40 lines and looks the same
  on both platforms. Keep the value as an **ISO `YYYY-MM-DD` string** (never a `Date` object) so it goes
  straight into a query string, re-seed the three columns from the incoming value in a
  `useEffect(..., [visible])` (so re-opening shows the current pick, not the last scroll), and clamp the
  day column to that month's length. Render one instance per bound (`from`, `to`) driven by a single
  `pick: 'from' | 'to' | null` state.
- **"Clear cache" = wipe AsyncStorage EXCEPT the session key.** A blanket `AsyncStorage.clear()` logs the
  user out, which reads as a crash-and-logout bug, not a cache clear. Read `getAllKeys()`, filter out the
  keep-list (session/user key, and anything the user chose like the language code), `multiRemove` the
  rest. Keep the keep-list as one exported constant so a new persisted key can't silently join the
  wipe set.

## ⚠️ A component declared INSIDE a component closes the keyboard on every keystroke

The highest-value RN anti-pattern that no lint rule catches, because the code looks like clean
extraction. A helper component defined in the render scope of its parent is a **new component type on
every render**, so React unmounts the old subtree and mounts a fresh one instead of re-rendering it. On
web that is merely slow; in RN a remounted `TextInput` loses focus, which **dismisses the keyboard the
instant the user types a character** — the field accepts exactly one digit per keyboard-open.

```tsx
// WRONG — `Field` is re-created each render; typing one character remounts the TextInput.
export default function RateScreen() {
  const [v, setV] = useState('');
  const Field = ({ label, value, onChange }: any) => (      // ← new TYPE every render
    <><Text>{label}</Text><TextInput value={value} onChangeText={onChange} /></>
  );
  return <Field label="Video call" value={v} onChange={setV} />;
}

// RIGHT — module scope. Identity is stable, so the TextInput re-renders and keeps focus.
function Field({ label, value, onChange }: any) { /* … */ }
export default function RateScreen() { /* … */ }
```

- **Symptom vocabulary to recognise in a bug report:** "the keyboard doesn't come up", "it closes as soon
  as I type", "I can only enter one number". Nobody reports it as a re-render problem, so it is usually
  chased in keyboard / `KeyboardAvoidingView` code where the cause is not.
- **It recurs.** Expect the same trap in every screen that grows an inline `Row`/`Field`/`Cell` helper —
  each one is an independent instance of the bug. When you fix one, sweep for a capitalised arrow
  assignment or a `function [A-Z]…` nested inside a component body and hoist all of them in the same pass
  rather than waiting for the next report.
- `useCallback` does **not** help: memoising the function does not make it a stable component *type*, and
  reaching for `useMemo` to dodge that is a worse version of just moving it to module scope. If the helper
  needs parent state, pass it as props — that is the whole reason the hoisted version takes
  `value` / `onChange`.
- Same root cause as the "inline style object recreated every render" anti-pattern above, but the
  consequence is a **functional** defect rather than a performance one, so it earns its own review check:
  when a `TextInput`'s focus is unstable, look at where its wrapper is declared before anything else.

## Related Skills

- `frontend-patterns` — React/Next.js (web) patterns; useful for shared React concepts, but DOM-specific.
- `coding-standards` — TypeScript/JavaScript idioms that apply to RN code.
- `tdd-workflow`, `e2e-testing` — testing process (use Jest + React Native Testing Library, Maestro/Detox for RN).
- `security-review` — general security checklist that complements the RN bundle/secret guidance above.
