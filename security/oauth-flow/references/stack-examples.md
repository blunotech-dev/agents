# Stack Examples

## Node.js + Express (server-side, confidential client)

```ts
import { Issuer, generators } from 'openid-client';

// Init once at startup
const issuer = await Issuer.discover('https://accounts.google.com');
const client = new issuer.Client({
  client_id: process.env.OAUTH_CLIENT_ID!,
  client_secret: process.env.OAUTH_CLIENT_SECRET!,
  redirect_uris: ['https://app.example.com/auth/callback'],
  response_types: ['code'],
});

// Login route
app.get('/auth/login', (req, res) => {
  const state = generators.state();
  const nonce = generators.nonce();
  const codeVerifier = generators.codeVerifier();
  const codeChallenge = generators.codeChallenge(codeVerifier);

  req.session.oauth = { state, nonce, codeVerifier };

  res.redirect(client.authorizationUrl({
    scope: 'openid profile email',
    state, nonce,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
  }));
});

// Callback route
app.get('/auth/callback', async (req, res) => {
  const { state, nonce, codeVerifier } = req.session.oauth ?? {};
  delete req.session.oauth;
  if (!state) return res.status(400).send('No pending session');

  const params = client.callbackParams(req);
  if (params.error) return res.status(400).send(params.error_description);

  const tokenSet = await client.callback(
    'https://app.example.com/auth/callback',
    params,
    { state, nonce, code_verifier: codeVerifier }
    // openid-client validates state, nonce, id_token automatically
  );

  const userInfo = tokenSet.claims();
  const user = await db.users.upsert({ where: { email: userInfo.email }, ... });
  req.session.userId = user.id;

  const returnTo = req.session.returnTo ?? '/dashboard';
  delete req.session.returnTo;
  res.redirect(isSameOrigin(returnTo) ? returnTo : '/dashboard');
});

function isSameOrigin(url: string) {
  try { return new URL(url, 'https://app.example.com').origin === 'https://app.example.com'; }
  catch { return false; }
}
```

---

## Python + FastAPI

```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

oauth = OAuth(Config('.env'))
oauth.register(
    name='google',
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_id=..., client_secret=...,
    client_kwargs={'scope': 'openid profile email', 'code_challenge_method': 'S256'},
)

@router.get('/auth/login')
async def login(request: Request):
    redirect_uri = str(request.url_for('auth_callback'))
    return await oauth.google.authorize_redirect(request, redirect_uri)
    # authlib handles state + PKCE automatically

@router.get('/auth/callback')
async def auth_callback(request: Request):
    token = await oauth.google.authorize_access_token(request)
    # validates state, PKCE, id_token automatically
    user_info = token['userinfo']
    user = await db.upsert_user(email=user_info['email'], name=user_info.get('name'))
    request.session['user_id'] = str(user.id)
    return_to = request.session.pop('return_to', '/dashboard')
    return RedirectResponse(return_to if is_same_origin(return_to) else '/dashboard')
```

---

## SPA (TypeScript, public client)

```ts
// oauth.ts — in-memory token store
let _token: string | null = null;
let _expiry = 0;

export function getToken() {
  return _token && Date.now() < _expiry - 60_000 ? _token : null;
}

export async function login(returnTo = location.pathname) {
  const verifier = rand(32);
  const challenge = await sha256base64url(verifier);
  const state = btoa(JSON.stringify({ n: rand(16), returnTo }));

  sessionStorage.setItem('pkce_v', verifier);
  sessionStorage.setItem('oauth_s', state);

  location.href = buildURL('https://accounts.google.com/o/oauth2/v2/auth', {
    response_type: 'code', client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI, scope: 'openid profile email',
    state, code_challenge: challenge, code_challenge_method: 'S256',
  });
}

export async function handleCallback() {
  const p = new URLSearchParams(location.search);
  if (p.get('error')) throw new Error(p.get('error_description')!);

  const state = p.get('state');
  const saved = sessionStorage.getItem('oauth_s');
  sessionStorage.removeItem('oauth_s');
  if (state !== saved) throw new Error('State mismatch');

  const verifier = sessionStorage.getItem('pkce_v')!;
  sessionStorage.removeItem('pkce_v');

  const res = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code', code: p.get('code')!,
      redirect_uri: REDIRECT_URI, client_id: CLIENT_ID, code_verifier: verifier,
    }),
  });
  if (!res.ok) throw new Error('Token exchange failed');

  const { access_token, expires_in } = await res.json();
  _token = access_token;
  _expiry = Date.now() + expires_in * 1000;

  return JSON.parse(atob(state!)).returnTo ?? '/';
}

// Helpers
const rand = (n: number) => base64url(crypto.getRandomValues(new Uint8Array(n)));
const encode = (s: string) => new TextEncoder().encode(s);
const base64url = (b: ArrayBuffer | Uint8Array) =>
  btoa(String.fromCharCode(...new Uint8Array(b as ArrayBuffer)))
    .replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
const sha256base64url = async (s: string) =>
  base64url(await crypto.subtle.digest('SHA-256', encode(s)));
const buildURL = (base: string, params: Record<string, string>) => {
  const u = new URL(base);
  Object.entries(params).forEach(([k,v]) => u.searchParams.set(k,v));
  return u.toString();
};
```