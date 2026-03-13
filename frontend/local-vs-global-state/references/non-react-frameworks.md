# Non-React Framework State Guidance

## Vue 3

| Category | Tool |
|---|---|
| Local | `ref()`, `reactive()` inside component |
| Shared (small subtree) | `provide` / `inject` |
| Global | Pinia store |
| Server | VueQuery (`@tanstack/vue-query`) or useFetch (Nuxt) |
| URL | `useRoute` / `useRouter` params and query |
| Form | VeeValidate, FormKit, or local `reactive()` |

**Key difference from React:** Vue's reactivity system makes local `reactive()` objects safe to pass around without hoisting. Only reach for Pinia when state is truly global or needs devtools/persistence.

---

## Svelte / SvelteKit

| Category | Tool |
|---|---|
| Local | `let` variable (auto-reactive in `.svelte`) |
| Shared (subtree) | Writable store passed as prop or context |
| Global | Writable/readable store in `$lib/stores` |
| Server | SvelteKit `load()` functions + page data |
| URL | `$page.url.searchParams`, `goto()` |
| Form | SvelteKit form actions, superforms |

**Key difference:** Svelte stores are just observables — a `writable()` can serve as both local and global state depending on where it's instantiated and exported.

---

## Angular

| Category | Tool |
|---|---|
| Local | Component property, `signal()` (v17+) |
| Shared (subtree) | `@Input()`/`@Output()`, or a scoped service |
| Global | Root-provided `Injectable` service with `signal()` or BehaviorSubject |
| Server | `HttpClient` in a service, `AsyncPipe` in templates |
| URL | `ActivatedRoute` params/queryParams |
| Form | `ReactiveFormsModule` (`FormControl`, `FormGroup`) |

**Key difference:** Angular's DI system makes "shared local" state natural — provide a service at the component level to scope it to a subtree rather than the whole app.

---

## Solid.js

| Category | Tool |
|---|---|
| Local | `createSignal()` |
| Shared (subtree) | `createContext()` + `useContext()` |
| Global | Store with `createStore()` or signals in a module |
| Server | `createResource()` |
| URL | `useSearchParams()` (SolidStart) |

Solid's fine-grained reactivity means Context doesn't cause excessive re-renders — it's safer to use Context more broadly than in React.