# FastAPI (Python) CRUD Patterns

## Setup assumptions
- FastAPI + Pydantic v2
- SQLAlchemy 2.x (async) or sync
- Alembic for migrations

## File layout
```
app/
  routers/entity.py
  schemas/entity.py
  models/entity.py
  crud/entity.py
```

## Pydantic schemas
```python
# schemas/entity.py
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class EntityBase(BaseModel):
    name: str
    description: Optional[str] = None

class EntityCreate(EntityBase):
    pass

class EntityUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None

class EntityOut(EntityBase):
    id: int
    created_at: datetime

    class Config:
        from_attributes = True
```

## Router
```python
# routers/entity.py
from fastapi import APIRouter, HTTPException, Depends
from sqlalchemy.orm import Session
from app import crud, schemas
from app.database import get_db

router = APIRouter(prefix="/entities", tags=["entities"])

@router.post("/", response_model=schemas.EntityOut, status_code=201)
def create(payload: schemas.EntityCreate, db: Session = Depends(get_db)):
    return crud.entity.create(db, payload)

@router.get("/", response_model=list[schemas.EntityOut])
def list_all(skip: int = 0, limit: int = 20, db: Session = Depends(get_db)):
    return crud.entity.get_all(db, skip=skip, limit=limit)

@router.get("/{id}", response_model=schemas.EntityOut)
def read(id: int, db: Session = Depends(get_db)):
    entity = crud.entity.get(db, id)
    if not entity:
        raise HTTPException(status_code=404, detail="Not found")
    return entity

@router.patch("/{id}", response_model=schemas.EntityOut)
def update(id: int, payload: schemas.EntityUpdate, db: Session = Depends(get_db)):
    entity = crud.entity.get(db, id)
    if not entity:
        raise HTTPException(status_code=404, detail="Not found")
    return crud.entity.update(db, entity, payload)

@router.delete("/{id}", status_code=204)
def delete(id: int, db: Session = Depends(get_db)):
    entity = crud.entity.get(db, id)
    if not entity:
        raise HTTPException(status_code=404, detail="Not found")
    crud.entity.delete(db, entity)
```

## Global exception handler (main.py)
```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_handler(request: Request, exc: Exception):
    return JSONResponse(status_code=500, content={"error": "Internal server error"})
```