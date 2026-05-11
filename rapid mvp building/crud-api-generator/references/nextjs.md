# Next.js App Router CRUD Patterns

## Setup assumptions
- Next.js 14+ App Router
- TypeScript
- Prisma ORM
- Route handlers in `app/api/`

## File layout
```
app/api/
  entities/
    route.ts          ← POST (create), GET (list)
  entities/[id]/
    route.ts          ← GET (read), PATCH (update), DELETE (delete)
```

## Collection route (app/api/entities/route.ts)
```ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    if (!body.name) {
      return NextResponse.json({ error: 'name is required' }, { status: 400 });
    }
    const entity = await prisma.entity.create({ data: body });
    return NextResponse.json({ data: entity }, { status: 201 });
  } catch (e) {
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}

export async function GET() {
  try {
    const entities = await prisma.entity.findMany();
    return NextResponse.json({ data: entities, total: entities.length });
  } catch (e) {
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

## Item route (app/api/entities/[id]/route.ts)
```ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

type Params = { params: { id: string } };

export async function GET(_req: NextRequest, { params }: Params) {
  const entity = await prisma.entity.findUnique({ where: { id: params.id } });
  if (!entity) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json({ data: entity });
}

export async function PATCH(req: NextRequest, { params }: Params) {
  const body = await req.json();
  try {
    const entity = await prisma.entity.update({ where: { id: params.id }, data: body });
    return NextResponse.json({ data: entity });
  } catch {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }
}

export async function DELETE(_req: NextRequest, { params }: Params) {
  try {
    await prisma.entity.delete({ where: { id: params.id } });
    return new NextResponse(null, { status: 204 });
  } catch {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }
}
```

## Notes
- For Pages Router, use `pages/api/entities/[id].ts` with method switching on `req.method`
- Auth: wrap handlers with `getServerSession` from `next-auth` or check cookies/headers