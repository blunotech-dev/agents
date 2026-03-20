---
name: e2e-auth-flow
description: Write E2E tests for full auth flows (register, login, logout, reset, protected routes) with session reuse. Use when testing authentication, session handling, or protected route access end-to-end.
category: "Testing"
---

# E2E Auth Flow Skill

## Discovery

Before writing tests, identify:

- **Framework** — Playwright (preferred) or Cypress; auth storage APIs differ significantly
- **Auth mechanism** — session cookie, JWT in localStorage, JWT in httpOnly cookie, OAuth/SSO
- **Token storage location** — determines how to capture and reuse state (`localStorage`, `sessionStorage`, `httpOnly` cookie)
- **Protected routes** — which pages require auth, which redirect to login when unauthenticated
- **Email flows** — does password reset require real email interception? (needs Mailhog, Mailtrap, or `+alias` trick)
- **Test user strategy** — seeded DB user, or register fresh each run?

---

## Session Reuse — The Core Pattern

Re-logging in before every test is the most common E2E auth mistake. It's slow and it couples every test to the login flow. Instead, authenticate once and save state:

### Playwright

```ts
// auth.setup.ts — runs once before the test suite
import { test as setup } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name=email]', process.env.TEST_USER_EMAIL!);
  await page.fill('[name=password]', process.env.TEST_USER_PASSWORD!);
  await page.click('[type=submit]');
  await page.waitForURL('/dashboard');

  // Save full browser state: cookies + localStorage + sessionStorage
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /auth\.setup\.ts/ },
    {
      name: 'authenticated',
      dependencies: ['setup'],
      use: { storageState: 'playwright/.auth/user.json' },
    },
  ],
});
```

### Cypress

```ts
// cypress/support/commands.ts
Cypress.Commands.add('loginByApi', () => {
  cy.request('POST', '/api/auth/login', {
    email: Cypress.env('TEST_USER_EMAIL'),
    password: Cypress.env('TEST_USER_PASSWORD'),
  }).then(({ body }) => {
    // Store token directly — skip the UI entirely
    window.localStorage.setItem('auth_token', body.token);
    // or cy.setCookie for cookie-based auth
  });
});
```

```ts
// In tests — never use cy.visit('/login') + fill form
beforeEach(() => cy.loginByApi());
```

API-based login in Cypress is 10-50x faster than UI login and doesn't flake on form animations.

---

## What to Test

### 1. Registration — assert the full side effect chain, not just redirect

```ts
test('registration creates account and sends verification email', async ({ page }) => {
  await page.goto('/register');
  await page.fill('[name=email]', uniqueEmail()); // generate unique to avoid conflicts
  await page.fill('[name=password]', 'SecurePass123!');
  await page.click('[type=submit]');

  // Assert redirect
  await expect(page).toHaveURL('/verify-email');

  // Assert DB side effect (if test DB accessible)
  const user = await db.findByEmail(email);
  expect(user.emailVerified).toBe(false);

  // Assert email was queued (check Mailhog or mock email service)
  const emails = await mailhog.getMessages({ to: email });
  expect(emails).toHaveLength(1);
  expect(emails[0].subject).toContain('Verify');
});
```

A test that only checks the redirect proves nothing about whether the account was actually created.

---

### 2. Login — test session persistence across page loads

```ts
test('session persists after page reload', async ({ page }) => {
  await login(page);
  await page.reload();
  // Must still be on authenticated page, not redirected to /login
  await expect(page).not.toHaveURL('/login');
  await expect(page.locator('[data-testid=user-menu]')).toBeVisible();
});
```

This catches bugs where auth state is stored in memory only and lost on reload.

---

### 3. Protected routes — test both the redirect AND the return

```ts
test('unauthenticated user is redirected to login with return URL', async ({ page }) => {
  // Access protected route without auth
  await page.goto('/dashboard/settings');
  await expect(page).toHaveURL(/\/login\?.*redirect/);

  // After login, must return to original destination
  await page.fill('[name=email]', testUser.email);
  await page.fill('[name=password]', testUser.password);
  await page.click('[type=submit]');
  await expect(page).toHaveURL('/dashboard/settings'); // not just /dashboard
});
```

Testing only the redirect misses the broken return URL — a very common bug.

---

### 4. Logout — assert session is actually destroyed, not just UI reset

```ts
test('logout invalidates session on server', async ({ page, request }) => {
  const storageState = await page.context().storageState();
  const cookies = storageState.cookies;

  await page.click('[data-testid=logout-button]');
  await expect(page).toHaveURL('/login');

  // Attempt API call with the old session cookie — must be rejected
  const res = await request.get('/api/me', {
    headers: { Cookie: cookies.map(c => `${c.name}=${c.value}`).join('; ') }
  });
  expect(res.status()).toBe(401); // session was actually invalidated server-side
});
```

Without this, a "logout" that only clears client state still leaves a valid session exploitable by anyone with the cookie.

---

### 5. Password reset — test token expiry, not just the happy path

```ts
// Happy path
test('valid reset token allows password change', async ({ page }) => {
  const token = await db.createPasswordResetToken(testUser.id, { expiresIn: '1h' });
  await page.goto(`/reset-password?token=${token}`);
  await page.fill('[name=password]', 'NewSecurePass456!');
  await page.click('[type=submit]');
  await expect(page).toHaveURL('/login');

  // Verify old password no longer works
  await loginWith(page, testUser.email, testUser.originalPassword);
  await expect(page).toHaveURL('/login'); // rejected
});

// Expired token
test('expired reset token shows error, not silent failure', async ({ page }) => {
  const token = await db.createPasswordResetToken(testUser.id, { expiresAt: pastDate() });
  await page.goto(`/reset-password?token=${token}`);
  await expect(page.locator('[data-testid=error]')).toContainText(/expired|invalid/i);
});

// Token reuse
test('reset token cannot be used twice', async ({ page }) => {
  const token = await db.createPasswordResetToken(testUser.id);
  await useResetToken(page, token, 'FirstNewPass123!');
  await page.goto(`/reset-password?token=${token}`);
  await expect(page.locator('[data-testid=error]')).toContainText(/expired|invalid/i);
});
```

---

### 6. Token refresh — test the silent refresh, not just initial auth

For JWT-based auth with refresh tokens:

```ts
test('expired access token is silently refreshed', async ({ page }) => {
  await login(page);

  // Expire the access token in storage without touching the refresh token
  await page.evaluate(() => {
    const auth = JSON.parse(localStorage.getItem('auth')!);
    auth.accessToken = 'expired.token.value';
    localStorage.setItem('auth', JSON.stringify(auth));
  });

  // Navigate to a protected page — should silently refresh, not redirect to login
  await page.goto('/dashboard');
  await expect(page).not.toHaveURL('/login');
  await expect(page.locator('[data-testid=user-menu]')).toBeVisible();
});
```

---

### 7. Concurrent sessions — test multi-tab behavior if relevant

```ts
test('logout in one tab invalidates other open sessions', async ({ browser }) => {
  const context = await browser.newContext({ storageState: authState });
  const tab1 = await context.newPage();
  const tab2 = await context.newPage();

  await tab1.goto('/dashboard');
  await tab2.goto('/dashboard');

  // Logout in tab1
  await tab1.click('[data-testid=logout-button]');

  // Navigate in tab2 — should be kicked out
  await tab2.goto('/dashboard');
  await expect(tab2).toHaveURL('/login');
});
```

---

## Test User Strategy

### Seed once, reuse — don't register via UI in every suite

```ts
// global-setup.ts
export default async function globalSetup() {
  await db.upsert('users', {
    email: process.env.TEST_USER_EMAIL,
    passwordHash: await hash(process.env.TEST_USER_PASSWORD),
    emailVerified: true,
  });
}
```

Registration via UI is for testing the registration flow specifically — not a setup utility.

### Use unique emails for registration tests

```ts
const uniqueEmail = () => `test+${Date.now()}@example.com`;
// or
const uniqueEmail = () => `test+${randomUUID()}@example.com`;
```

Static test emails in registration tests cause conflicts when tests run in parallel or are re-run without DB cleanup.

---

## What Not to Do

- **Don't use UI login in `beforeEach`** — save and reuse storage state; UI login in setup is only for the auth setup step itself
- **Don't hardcode credentials** — always use `process.env` / `Cypress.env`; never commit test passwords
- **Don't test only the redirect on protected routes** — test the return URL too
- **Don't trust client-side logout** — always verify the session is invalidated server-side with a raw API call
- **Don't skip token expiry and reuse tests for password reset** — these are the cases attackers actually target
- **Don't use `page.waitForTimeout`** — wait for URL, element visibility, or network idle instead; timeouts are flaky

---

## Output Format

Group tests into four `describe` blocks: registration, login/session, protected routes, and account recovery (password reset). Auth setup lives in a dedicated setup file, not in `beforeEach`. Each test is independent of execution order — no test should depend on another test having run first.