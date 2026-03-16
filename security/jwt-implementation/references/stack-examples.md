# Stack-Specific JWT Implementation Examples

## Node.js + Express (TypeScript)

### Dependencies
```bash
npm install jsonwebtoken bcryptjs redis uuid
npm install -D @types/jsonwebtoken @types/bcryptjs @types/uuid
```

### Token generation

```ts
import jwt from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';
import crypto from 'crypto';
import { redis } from './redis'; // ioredis client
import { db } from './db';      // your DB client

const ACCESS_SECRET = process.env.JWT_PRIVATE_KEY!; // RS256 private key
const ACCESS_PUBLIC = process.env.JWT_PUBLIC_KEY!;
const ACCESS_EXPIRY = 15 * 60; // 15 minutes in seconds

export function issueAccessToken(userId: string, roles: string[]): string {
  return jwt.sign(
    { sub: userId, roles, jti: uuidv4() },
    ACCESS_SECRET,
    { algorithm: 'RS256', expiresIn: ACCESS_EXPIRY }
  );
}

export async function issueRefreshToken(userId: string, familyId: string): Promise<string> {
  const raw = crypto.randomBytes(64).toString('hex');
  const hash = crypto.createHash('sha256').update(raw).digest('hex');

  await db.refreshTokens.create({
    data: {
      family_id: familyId,
      token_hash: hash,
      user_id: userId,
      expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30d
    }
  });

  return raw;
}
```

### Token verification middleware

```ts
import { Request, Response, NextFunction } from 'express';

export async function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.cookies.access_token || req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });

  let payload: jwt.JwtPayload;
  try {
    payload = jwt.verify(token, ACCESS_PUBLIC, {
      algorithms: ['RS256'],
      clockTolerance: 30, // 30s skew tolerance
    }) as jwt.JwtPayload;
  } catch (e) {
    return res.status(401).json({ error: 'Invalid token' });
  }

  // Check blocklist
  const blocked = await redis.get(`blocklist:${payload.jti}`);
  if (blocked) return res.status(401).json({ error: 'Token revoked' });

  req.user = { id: payload.sub!, roles: payload.roles };
  next();
}
```

### Refresh endpoint with rotation

```ts
app.post('/auth/refresh', async (req, res) => {
  const raw = req.cookies.refresh_token;
  if (!raw) return res.status(401).json({ error: 'No refresh token' });

  const hash = crypto.createHash('sha256').update(raw).digest('hex');
  const stored = await db.refreshTokens.findUnique({ where: { token_hash: hash } });

  if (!stored || stored.revoked || stored.expires_at < new Date()) {
    // If token was already revoked → replay detected → revoke entire family
    if (stored?.revoked) {
      await db.refreshTokens.updateMany({
        where: { family_id: stored.family_id },
        data: { revoked: true }
      });
    }
    return res.status(401).json({ error: 'Invalid refresh token' });
  }

  // Rotate: revoke old, issue new
  await db.refreshTokens.update({ where: { id: stored.id }, data: { revoked: true } });

  const accessToken = issueAccessToken(stored.user_id, ['user']); // fetch roles from DB in prod
  const newRefresh = await issueRefreshToken(stored.user_id, stored.family_id);

  res.cookie('access_token', accessToken, {
    httpOnly: true, secure: true, sameSite: 'strict', maxAge: 15 * 60 * 1000
  });
  res.cookie('refresh_token', newRefresh, {
    httpOnly: true, secure: true, sameSite: 'strict',
    maxAge: 30 * 24 * 60 * 60 * 1000, path: '/auth/refresh'
  });

  res.json({ ok: true });
});
```

### Logout

```ts
app.post('/auth/logout', requireAuth, async (req, res) => {
  const token = req.cookies.access_token;
  const payload = jwt.decode(token) as jwt.JwtPayload;

  // Blocklist the access token for its remaining TTL
  const ttl = payload.exp! - Math.floor(Date.now() / 1000);
  if (ttl > 0) await redis.set(`blocklist:${payload.jti}`, '1', 'EX', ttl);

  // Revoke all refresh tokens for this family (or just the current one)
  const raw = req.cookies.refresh_token;
  if (raw) {
    const hash = crypto.createHash('sha256').update(raw).digest('hex');
    const stored = await db.refreshTokens.findUnique({ where: { token_hash: hash } });
    if (stored) {
      await db.refreshTokens.updateMany({
        where: { family_id: stored.family_id },
        data: { revoked: true }
      });
    }
  }

  res.clearCookie('access_token');
  res.clearCookie('refresh_token');
  res.json({ ok: true });
});
```

---

## Python + FastAPI

### Dependencies
```bash
pip install python-jose[cryptography] passlib[bcrypt] redis fastapi
```

### Token utils

```python
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone
import secrets, hashlib, uuid

ALGORITHM = "RS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15

def create_access_token(user_id: str, roles: list[str]) -> str:
    jti = str(uuid.uuid4())
    expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {"sub": user_id, "roles": roles, "jti": jti, "exp": expire}
    return jwt.encode(payload, PRIVATE_KEY, algorithm=ALGORITHM)

def verify_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, PUBLIC_KEY, algorithms=[ALGORITHM])
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

    jti = payload.get("jti")
    if redis_client.get(f"blocklist:{jti}"):
        raise HTTPException(status_code=401, detail="Token revoked")

    return payload

def create_refresh_token(user_id: str, family_id: str) -> str:
    raw = secrets.token_hex(64)
    token_hash = hashlib.sha256(raw.encode()).hexdigest()
    db.execute(
        "INSERT INTO refresh_tokens (family_id, token_hash, user_id, expires_at) VALUES (%s, %s, %s, %s)",
        (family_id, token_hash, user_id, datetime.now(timezone.utc) + timedelta(days=30))
    )
    return raw
```

### Auth dependency

```python
from fastapi import Depends, Cookie, HTTPException

async def get_current_user(access_token: str = Cookie(None)):
    if not access_token:
        raise HTTPException(status_code=401)
    return verify_access_token(access_token)
```

---

## Go + net/http

### Dependencies
```bash
go get github.com/golang-jwt/jwt/v5
go get github.com/redis/go-redis/v9
```

### Token generation

```go
import (
    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
    "crypto/rsa"
    "time"
)

type Claims struct {
    Roles []string `json:"roles"`
    jwt.RegisteredClaims
}

func IssueAccessToken(userID string, roles []string, privateKey *rsa.PrivateKey) (string, error) {
    claims := Claims{
        Roles: roles,
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   userID,
            ID:        uuid.NewString(),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
        },
    }
    return jwt.NewWithClaims(jwt.SigningMethodRS256, claims).SignedString(privateKey)
}

func VerifyAccessToken(tokenStr string, publicKey *rsa.PublicKey) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return publicKey, nil
    })
    if err != nil || !token.Valid {
        return nil, err
    }
    return token.Claims.(*Claims), nil
}
```