1. file and folder structure
2. database connection === tables register, queries 
3. schemas == input uuser sy usko validate 
4. functions === code reuse
5. route and api create

pip install fastapi uvicorn sqlalchemy psycopg2-binary alembic pydantic python-dotenv

project/
│
├── alembic/
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
│
├── app/
│   │
│   ├── api/
│   │   └── todo.py
│   │
│   ├── core/
│   │   └── database.py, config.py
│   │
│   ├── models/
│   │   └── todo.py
│   │
│   ├── schemas/
│   │   └── todo.py
│   │
│   ├── crud/
│   │   └── todo.py
│   │
│   └── __init__.py
│
├── main.py
│
├── alembic.ini
├── requirements.txt
└── .env

create database todo_db;

1. .env
DB_USER=root
DB_PASSWORD=123456
DB_HOST=localhost
DB_PORT=3306
DB_NAME=todo_db

2. alembic/env.py

from logging.config import fileConfig
from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic import context

from dotenv import load_dotenv
import os

load_dotenv()

DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")
DB_PORT = os.getenv("DB_PORT")
DB_NAME = os.getenv("DB_NAME")

DATABASE_URL = (
    f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}"
    f"@{DB_HOST}:{DB_PORT}/{DB_NAME}"
)

config = context.config

config.set_main_option(
    "sqlalchemy.url",
    DATABASE_URL
)

from app.core.database import Base
from app.models.todo import Todo

target_metadata = Base.metadata

3 . alembic.ini

sqlalchemy.url =


4. app/core/config.py

from dotenv import load_dotenv
import os

load_dotenv()

DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")
DB_PORT = os.getenv("DB_PORT")
DB_NAME = os.getenv("DB_NAME")

DATABASE_URL = (
    f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}"
    f"@{DB_HOST}:{DB_PORT}/{DB_NAME}"
)

5. app/core/database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.orm import declarative_base

from app.core.config import DATABASE_URL

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

Base = declarative_base()


def get_db():
    db = SessionLocal()

    try:
        yield db

    finally:
        db.close()


6. app/models/todo.py 
from sqlalchemy import Column, Integer, String
from app.core.database import Base


class Todo(Base):
    __tablename__ = "todos"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False)
    description = Column(String, nullable=True)

alembic revision --autogenerate -m "create todo table"
alembic upgrade head


7. app/schemas/todo.py

from pydantic import BaseModel


class TodoCreate(BaseModel):
    title: str
    description: str | None = None


class TodoResponse(BaseModel):
    id: int
    title: str
    description: str | None = None

    class Config:
        from_attributes = True

8. app/crud/todo.py

from sqlalchemy.orm import Session
from app.models.todo import Todo
from app.schemas.todo import TodoCreate


def create_todo(db: Session, todo: TodoCreate):

    db_todo = Todo(
        title=todo.title,
        description=todo.description
    )

    db.add(db_todo)
    db.commit()
    db.refresh(db_todo)

    return db_todo


def get_todos(db: Session):
    return db.query(Todo).all()


def get_todo(db: Session, todo_id: int):
    return db.query(Todo).filter(
        Todo.id == todo_id
    ).first()


def update_todo(
    db: Session,
    todo_id: int,
    todo: TodoCreate
):
    db_todo = get_todo(db, todo_id)

    if not db_todo:
        return None

    db_todo.title = todo.title
    db_todo.description = todo.description

    db.commit()
    db.refresh(db_todo)

    return db_todo


def delete_todo(
    db: Session,
    todo_id: int
):
    db_todo = get_todo(db, todo_id)

    if not db_todo:
        return None

    db.delete(db_todo)
    db.commit()

    return db_todo

9. app/api/todo.py

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.core.database import get_db
from app.schemas.todo import (
    TodoCreate,
    TodoResponse
)
from app.crud.todo import (
    create_todo,
    get_todos,
    get_todo,
    update_todo,
    delete_todo
)

router = APIRouter(
    prefix="/todos",
    tags=["Todos"]
)


@router.post(
    "/",
    response_model=TodoResponse
)
def create(
    todo: TodoCreate,
    db: Session = Depends(get_db)
):
    return create_todo(db, todo)


@router.get(
    "/",
    response_model=list[TodoResponse]
)
def read_all(
    db: Session = Depends(get_db)
):
    return get_todos(db)


@router.get(
    "/{todo_id}",
    response_model=TodoResponse
)
def read_one(
    todo_id: int,
    db: Session = Depends(get_db)
):
    todo = get_todo(db, todo_id)

    if not todo:
        raise HTTPException(
            status_code=404,
            detail="Todo not found"
        )

    return todo


@router.put(
    "/{todo_id}",
    response_model=TodoResponse
)
def update(
    todo_id: int,
    todo: TodoCreate,
    db: Session = Depends(get_db)
):
    updated = update_todo(
        db,
        todo_id,
        todo
    )

    if not updated:
        raise HTTPException(
            status_code=404,
            detail="Todo not found"
        )

    return updated


@router.delete("/{todo_id}")
def delete(
    todo_id: int,
    db: Session = Depends(get_db)
):
    deleted = delete_todo(
        db,
        todo_id
    )

    if not deleted:
        raise HTTPException(
            status_code=404,
            detail="Todo not found"
        )

    return {
        "message": "Todo deleted successfully"
    }

10. main.py

from fastapi import FastAPI

from app.api.todo import router as todo_router

app = FastAPI(
    title="Todo API"
)

app.include_router(todo_router)


@app.get("/")
def home():
    return {
        "message": "FastAPI Running"
    }   