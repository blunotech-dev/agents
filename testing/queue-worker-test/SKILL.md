---
name: queue-worker-test
description: Write tests for background jobs and queue workers covering processing logic, retries, and failure handling. Use when testing workers, job processors, or queue consumers and their behavior.
category: "Testing"
---

# Queue Worker Test

Generates tests for background job processors — verifying job execution, retry semantics, failure handling, and side effects without depending on a live queue broker.

---

## Phase 1: Discovery

Extract before writing:

1. **Queue library** — BullMQ, Bull (v3), SQS consumer, custom polling loop, or other?
2. **Test runner** — Jest or Vitest?
3. **What the worker does** — what side effects does a successful job produce? (DB write, HTTP call, email, etc.)
4. **Retry config** — how many attempts? backoff strategy? Is it library-managed or custom?
5. **Failure behavior** — does the worker move jobs to a dead-letter queue, emit an event, or just log?
6. **Concurrency** — does the worker process jobs in parallel? (affects test isolation approach)
7. **Existing test infrastructure** — is there already a DB test setup, mock HTTP layer, or job factory?

---

## Phase 2: Architecture Decisions

### Do not use a real queue broker in tests

A live Redis/SQS in tests creates flakiness (broker availability, leftover jobs from prior runs, timing). Instead:

| Approach | When to use |
|---|---|
| Call the processor function directly | Best default — bypasses queue entirely, tests the logic |
| In-memory queue (BullMQ `ioredis-mock`) | When you need to test worker registration, concurrency, or queue event hooks |
| Mocked queue client | Only when testing the code that *enqueues* jobs, not the processor |

**Default strategy:** extract the processor function so it can be called independently of the queue framework. If the user's worker is tightly coupled to the queue client (e.g., calls `job.moveToFailed()` directly), note this as a testability issue and mock only those queue calls.

### Processor function isolation

The processor should be extractable:

```ts
// worker.ts
export async function processEmailJob(job: Job<EmailJobData>): Promise<void> {
  // logic here — this is what we test
}

worker.process(processEmailJob) // registration — not tested directly
```

If the user's codebase doesn't separate these, generate the test by constructing a minimal fake `Job` object rather than mocking the entire queue.

---

## Phase 3: Test Case Coverage

### Job pickup (when testing queue integration)

Only test pickup if the user needs to verify worker registration or concurrency config. For most cases, skip this — it tests the library, not the user's code.

When needed:

```ts
it('picks up a queued job', async () => {
  const worker = new Worker('email', processEmailJob, { connection })
  await queue.add('email', { to: 'a@b.com', subject: 'Hi' })
  await new Promise(res => worker.on('completed', res)) // event-driven, not sleep()
  await worker.close()
})
```

**Non-obvious:** never use `setTimeout` or `sleep()` to wait for job completion — it produces timing-dependent failures in CI. Use the queue's completion event or poll with a proper retry utility.

### Processing logic tests

Test the processor function directly. Construct a fake `Job` object — only stub the properties the processor actually uses:

```ts
function makeJob(data: Partial<EmailJobData>, opts: Partial<Job> = {}): Job<EmailJobData> {
  return {
    id: '1',
    data: { to: 'test@example.com', subject: 'Test', ...data },
    attemptsMade: 0,
    log: jest.fn(),
    updateProgress: jest.fn(),
    ...opts,
  } as unknown as Job<EmailJobData>
}
```

Cover:
- **Happy path** — job completes, side effects occur, no errors thrown
- **Data variants** — optional fields absent, boundary values (empty string, max-length content)
- **Side effect assertions** — if the job sends an email, assert the mailer was called with the right args; if it writes to DB, query the DB directly

### Retry behavior

**Non-obvious:** retry logic lives in two places — the queue config (`attempts`, `backoff`) and sometimes inside the processor itself (manual retry on specific errors). Test them separately.

For library-managed retries, don't test that BullMQ retries — test that your processor **throws the right error type** so the library knows to retry:

```ts
it('throws on transient DB error so the job retries', async () => {
  jest.spyOn(db, 'insert').mockRejectedValueOnce(new Error('deadlock detected'))
  const job = makeJob({})
  await expect(processEmailJob(job)).rejects.toThrow('deadlock')
})
```

For custom retry logic inside the processor:

```ts
it('retries the HTTP call up to 3 times before failing', async () => {
  const spy = jest.spyOn(httpClient, 'post')
    .mockRejectedValueOnce(new Error('timeout'))
    .mockRejectedValueOnce(new Error('timeout'))
    .mockResolvedValueOnce({ status: 200 })

  await processEmailJob(makeJob({}))
  expect(spy).toHaveBeenCalledTimes(3)
})
```

### Failure handling

Three distinct failure cases to cover:

**1. Expected business failure** — the job data is valid but the operation can't complete (user deleted, resource gone). Should NOT retry:

```ts
it('does not rethrow on expected business failure', async () => {
  jest.spyOn(userRepo, 'findById').mockResolvedValueOnce(null)
  const job = makeJob({ userId: 'deleted-user' })
  await expect(processEmailJob(job)).resolves.not.toThrow()
  // processor swallowed the error deliberately
})
```

**2. Transient failure** — network timeout, DB deadlock. SHOULD retry (throws):

```ts
it('propagates transient errors for retry', async () => {
  jest.spyOn(emailService, 'send').mockRejectedValueOnce(new NetworkError('timeout'))
  await expect(processEmailJob(makeJob({}))).rejects.toBeInstanceOf(NetworkError)
})
```

**3. Dead-letter / exhausted retries** — what happens when `attemptsMade >= maxAttempts`:

```ts
it('emits failure event when attempts exhausted', async () => {
  const onFailed = jest.fn()
  jest.spyOn(emailService, 'send').mockRejectedValue(new Error('always fails'))
  const job = makeJob({}, { attemptsMade: 3 }) // at max
  
  await expect(processEmailJob(job)).rejects.toThrow()
  // if the processor itself handles exhaustion:
  expect(alertService.notify).toHaveBeenCalledWith(expect.objectContaining({ jobId: '1' }))
})
```

**Non-obvious:** if the processor calls `job.moveToFailed()` directly (BullMQ pattern), mock that method and assert it was called — don't let it actually move the job, which requires a live Redis connection.

### Concurrency and ordering

Only generate these tests if the worker explicitly handles concurrency (parallel jobs, mutex locks, deduplication):

```ts
it('does not process the same userId concurrently', async () => {
  const lock = new Map()
  const results = await Promise.all([
    processEmailJob(makeJob({ userId: 'u1' })),
    processEmailJob(makeJob({ userId: 'u1' })), // same user
  ])
  // assert idempotency or mutex behavior here
})
```

Skip this section if concurrency is managed entirely by the queue library config.

---

## Phase 4: Test Structure

```ts
describe('EmailWorker', () => {
  describe('processing', () => { /* happy path, data variants */ })
  describe('retry behavior', () => { /* transient errors, custom retry */ })
  describe('failure handling', () => { /* business failures, exhaustion */ })
  describe('side effects', () => { /* DB writes, HTTP calls, events emitted */ })
})
```

**Setup pattern:**

```ts
let mailerSpy: jest.SpyInstance

beforeEach(() => {
  mailerSpy = jest.spyOn(mailer, 'send').mockResolvedValue({ messageId: 'test-id' })
})

afterEach(() => {
  jest.resetAllMocks()
})
```

Do NOT use `jest.mock()` module-level mocks for the mailer/DB if you need per-test return value control — `jest.spyOn` in `beforeEach` gives per-test flexibility without the hoisting complexity.

---

## Phase 5: Output

Produce:

1. **Test file** — processor called directly, fake `Job` factory, all four coverage areas
2. **`makeJob` helper** — typed, accepts data and option overrides
3. **Notes on any testability issues** — if the processor is tightly coupled to the queue client, flag the methods that need mocking and suggest the extraction refactor

### Output notes

- If the user's processor calls `job.log()` or `job.updateProgress()`, include those in `makeJob` as `jest.fn()` — they'll throw otherwise
- Never `await new Promise(res => setTimeout(res, 500))` — use events or proper async utilities
- If Vitest: `vi.spyOn`, `vi.fn()`, `vi.resetAllMocks()` — otherwise identical
- For SQS consumers: the unit of test is the message handler function, not the polling loop — same direct-call strategy applies