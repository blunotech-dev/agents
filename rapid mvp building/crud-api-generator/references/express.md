# Express (JS/TS) CRUD Patterns

## Setup assumptions
- Express 4.x or 5.x
- TypeScript preferred; omit types for plain JS
- Prisma ORM (swap with your ORM of choice)
- `express-async-errors` or manual try/catch

## File layout
```
src/
  routes/entity.routes.ts
  controllers/entity.controller.ts
  services/entity.service.ts
  middleware/validate.ts
```

## Router
```ts
// routes/entity.routes.ts
import { Router } from 'express';
import * as ctrl from '../controllers/entity.controller';

const router = Router();

router.post('/', ctrl.create);
router.get('/', ctrl.list);
router.get('/:id', ctrl.read);
router.patch('/:id', ctrl.update);
router.delete('/:id', ctrl.remove);

export default router;
```

## Controller pattern
```ts
// controllers/entity.controller.ts
import { Request, Response } from 'express';
import * as service from '../services/entity.service';

export const create = async (req: Request, res: Response) => {
  const { name, ...rest } = req.body;
  if (!name) return res.status(400).json({ error: 'name is required' });

  const entity = await service.create({ name, ...rest });
  res.status(201).json({ data: entity });
};

export const read = async (req: Request, res: Response) => {
  const entity = await service.findById(req.params.id);
  if (!entity) return res.status(404).json({ error: 'Not found' });
  res.json({ data: entity });
};

export const list = async (_req: Request, res: Response) => {
  const [entities, total] = await service.findAll();
  res.json({ data: entities, total });
};

export const update = async (req: Request, res: Response) => {
  const entity = await service.findById(req.params.id);
  if (!entity) return res.status(404).json({ error: 'Not found' });

  const updated = await service.update(req.params.id, req.body);
  res.json({ data: updated });
};

export const remove = async (req: Request, res: Response) => {
  const entity = await service.findById(req.params.id);
  if (!entity) return res.status(404).json({ error: 'Not found' });

  await service.remove(req.params.id);
  res.status(204).send();
};
```

## Zod validation (optional)
```ts
import { z } from 'zod';

export const createSchema = z.object({
  name: z.string().min(1),
  // add fields
});

// In controller:
const parsed = createSchema.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ error: parsed.error.flatten() });
```

## Error handler middleware (add once in app.ts)
```ts
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
});
```