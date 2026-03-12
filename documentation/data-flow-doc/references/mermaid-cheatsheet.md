# Mermaid Cheatsheet for Data Flow Diagrams

## sequenceDiagram — for request/response flows

```mermaid
sequenceDiagram
  actor User
  participant A as Component A
  participant B as Service B

  User->>A: action (sync call)
  A->>B: request
  B-->>A: response (dashed = return)
  A-->>User: result

  Note over A,B: You can add notes

  alt Condition true
    A->>B: branch 1
  else Condition false
    A->>B: branch 2
  end

  loop Retry up to 3 times
    A->>B: retry request
  end

  opt Optional step
    A->>A: do something optional
  end
```

**Arrow types:**
- `A->>B: msg` — solid arrow (call / send)
- `A-->>B: msg` — dashed arrow (return / response)
- `A-)B: msg` — async (fire and forget)
- `A-xB: msg` — message that destroys target

**Activation boxes** (show when something is "active"):
```
A->>+B: start
B-->>-A: end
```

---

## flowchart TD — for branching / decision flows

```mermaid
flowchart TD
  A([Start]) --> B[Process Step]
  B --> C{Decision?}
  C -- Yes --> D[Branch A]
  C -- No --> E[Branch B]
  D --> F([End])
  E --> F

  subgraph "Sub-system"
    G[Step 1] --> H[Step 2]
  end
```

**Node shapes:**
- `[Text]` — rectangle (process)
- `{Text}` — diamond (decision)
- `(Text)` — rounded rectangle
- `([Text])` — stadium / pill (start/end)
- `[[Text]]` — subroutine
- `[(Text)]` — cylinder (database)
- `>Text]` — flag (annotation)

**Edge types:**
- `A --> B` — arrow
- `A --- B` — line, no arrow
- `A -- label --> B` — labelled arrow
- `A -.-> B` — dashed arrow
- `A ==> B` — thick arrow

**Direction:**
- `TD` / `TB` — top to bottom
- `LR` — left to right
- `BT` — bottom to top
- `RL` — right to left

---

## stateDiagram-v2 — for state machine flows

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> Loading : submit()
  Loading --> Success : response OK
  Loading --> Error : response fail
  Error --> Idle : reset()
  Success --> [*]

  state Loading {
    [*] --> Validating
    Validating --> Sending
  }
```

---

## Common Patterns

### HTTP Request/Response
```mermaid
sequenceDiagram
  participant Client
  participant API
  participant DB

  Client->>+API: POST /resource {body}
  API->>API: Validate + Auth
  API->>+DB: INSERT ...
  DB-->>-API: { id }
  API-->>-Client: 201 Created { id }
```

### Optimistic UI Update
```mermaid
sequenceDiagram
  participant User
  participant Store
  participant API

  User->>Store: dispatch(action)
  Store->>Store: optimistic update
  Store-->>User: UI updates immediately
  Store->>API: POST request (background)
  alt Success
    API-->>Store: confirmed
  else Failure
    API-->>Store: error
    Store->>Store: rollback
    Store-->>User: show error
  end
```

### Pub/Sub / Event Queue
```mermaid
flowchart LR
  Producer -->|publish event| Queue[(Queue)]
  Queue -->|consume| ConsumerA
  Queue -->|consume| ConsumerB
  ConsumerA --> ServiceA
  ConsumerB --> ServiceB
```

### Auth Middleware
```mermaid
flowchart TD
  Req([Incoming Request]) --> MW{Auth Middleware}
  MW -- No token --> 401([401 Unauthorized])
  MW -- Invalid token --> 401
  MW -- Valid token --> Handler[Route Handler]
  Handler --> Res([Response])
```