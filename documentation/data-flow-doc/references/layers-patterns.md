# Layer Naming Patterns by Stack

Use these canonical layer names when documenting flows in specific stacks.
Consistency across documents in the same codebase matters more than following
these exactly — if a codebase has its own naming convention, use that.

---

## React (SPA)

| Layer | Canonical Name |
|---|---|
| User gesture | `User` |
| Component | `[ComponentName] Component` |
| Custom hook | `use[Name] Hook` |
| Context | `[Name] Context` |
| State store | `[Store] Store` (Zustand / Redux / Jotai) |
| API call | `[Resource] API Client` |
| Utility / helper | `[name]Utils` |

**Typical flow direction:**
`User → Component → Hook → Store → API Client → (server)`

---

## Next.js (App Router)

| Layer | Canonical Name |
|---|---|
| User | `User` |
| Client Component | `[Name] Client Component` |
| Server Component | `[Name] Server Component` |
| Server Action | `[name] Server Action` |
| Route Handler | `[METHOD] /api/[route]` |
| Middleware | `Next.js Middleware` |
| Service | `[Name] Service` |
| Data Access | `[Name] Repository` |
| ORM | `Prisma` / `Drizzle` |
| Database | `[Postgres / SQLite / etc.]` |
| Cache | `Next.js Cache` / `Redis` |

**Two distinct flow types in Next.js:**
1. Server Component fetch: `User → Server Component → Service → DB`
2. Client mutation: `User → Client Component → Server Action → Service → DB`

---

## Next.js (Pages Router)

| Layer | Canonical Name |
|---|---|
| User | `User` |
| Page Component | `[Name] Page` |
| API Route | `[METHOD] /api/[path]` |
| Service | `[Name] Service` |
| Repository | `[Name] Repository` |
| Database | `[DB name]` |

---

## Express / Node.js API

| Layer | Canonical Name |
|---|---|
| HTTP Client | `HTTP Client` |
| Router | `Express Router` |
| Middleware | `[Name] Middleware` (auth, validation, logging) |
| Controller | `[Name] Controller` |
| Service | `[Name] Service` |
| Repository | `[Name] Repository` |
| ORM | `Prisma` / `TypeORM` / `Sequelize` |
| Database | `[DB name]` |
| Cache | `Redis` |
| Queue | `[BullMQ / SQS / RabbitMQ]` |

---

## Ruby on Rails

| Layer | Canonical Name |
|---|---|
| Browser | `Browser` |
| Router | `Rails Router` |
| Controller | `[Name]Controller` |
| Before Action | `before_action :[name]` |
| Model | `[Name] Model` |
| Concern | `[Name] Concern` |
| Service Object | `[Name] Service` |
| Query Object | `[Name] Query` |
| Job | `[Name] Job (ActiveJob)` |
| Mailer | `[Name] Mailer` |
| Database | `ActiveRecord / [DB name]` |
| Cache | `Rails Cache / Redis` |

---

## Django / Python

| Layer | Canonical Name |
|---|---|
| Client | `Client` |
| URL Router | `urls.py` |
| Middleware | `[Name] Middleware` |
| View | `[Name] View` |
| Serializer | `[Name] Serializer (DRF)` |
| Service | `[Name] Service` |
| Model | `[Name] Model` |
| Manager | `[Name] Manager` |
| Database | `Django ORM / [DB name]` |
| Cache | `Django Cache / Redis` |
| Task | `[Name] Task (Celery)` |

---

## Go (standard HTTP / chi / Echo)

| Layer | Canonical Name |
|---|---|
| Client | `HTTP Client` |
| Router | `[chi / Echo / net/http] Router` |
| Middleware | `[Name] Middleware` |
| Handler | `[Name] Handler` |
| Service | `[Name] Service` |
| Repository | `[Name] Repository` |
| Database | `database/sql / GORM / [DB name]` |
| Cache | `Redis / in-memory cache` |

---

## GraphQL (any backend)

| Layer | Canonical Name |
|---|---|
| Client | `GraphQL Client (Apollo / urql)` |
| Gateway | `GraphQL Gateway` |
| Resolver | `[Type].[field] Resolver` |
| DataLoader | `[Name] DataLoader` |
| Service | `[Name] Service` |
| Database | `[DB name]` |

---

## Event-Driven / Message Queue

| Layer | Canonical Name |
|---|---|
| Publisher | `[Name] Publisher` |
| Queue / Topic | `[QueueName] Queue` / `[TopicName] Topic` |
| Consumer | `[Name] Consumer` |
| Dead Letter | `[Name] DLQ` |
| Handler | `[Name] Handler` |

---

## Mobile (React Native / Flutter)

| Layer | Canonical Name |
|---|---|
| User | `User` |
| Screen | `[Name] Screen` |
| Component | `[Name] Component` |
| State / Store | `[Name] Store` |
| API Client | `[Name] API Client` |
| Local DB | `SQLite / MMKV / AsyncStorage` |
| Push Notification | `[FCM / APNs] Push` |

---

## Notes

- **Be specific over generic.** "Backend" is not a layer. "Express Controller" is.
- **Don't invent layers.** If a codebase has no service layer, don't add one to the diagram.
- **Abbreviate consistently.** If you write `UserRepo` once, use it throughout.
- **For monorepos**, prefix with the package: `web/useAuth Hook`, `api/AuthService`.