import os

project_name = "workout_api"
app_path = os.path.join(project_name, "app")

files = {
    f"{project_name}/requirements.txt": """fastapi
uvicorn
sqlalchemy
pydantic
fastapi-pagination
""",

    f"{app_path}/__init__.py": """# Pacote app""",

    f"{app_path}/database.py": """from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
""",

    f"{app_path}/models.py": """from sqlalchemy import Column, Integer, String
from .database import Base

class Atleta(Base):
    __tablename__ = "atletas"

    id = Column(Integer, primary_key=True, index=True)
    nome = Column(String, index=True)
    cpf = Column(String, unique=True, index=True)
    centro_treinamento = Column(String)
    categoria = Column(String)
""",

    f"{app_path}/schemas.py": """from pydantic import BaseModel

class AtletaBase(BaseModel):
    nome: str
    cpf: str
    centro_treinamento: str
    categoria: str

class AtletaCreate(AtletaBase):
    pass

class AtletaResponse(BaseModel):
    nome: str
    centro_treinamento: str
    categoria: str

    class Config:
        orm_mode = True
""",

    f"{app_path}/crud.py": """from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from fastapi import HTTPException
from . import models, schemas

def get_atletas(db: Session, skip: int = 0, limit: int = 10, nome: str = None, cpf: str = None):
    query = db.query(models.Atleta)
    if nome:
        query = query.filter(models.Atleta.nome.contains(nome))
    if cpf:
        query = query.filter(models.Atleta.cpf == cpf)
    return query.offset(skip).limit(limit).all()

def create_atleta(db: Session, atleta: schemas.AtletaCreate):
    db_atleta = models.Atleta(
        nome=atleta.nome,
        cpf=atleta.cpf,
        centro_treinamento=atleta.centro_treinamento,
        categoria=atleta.categoria,
    )
    try:
        db.add(db_atleta)
        db.commit()
        db.refresh(db_atleta)
        return db_atleta
    except IntegrityError:
        db.rollback()
        raise HTTPException(
            status_code=303,
            detail=f"Já existe um atleta cadastrado com o cpf: {atleta.cpf}",
        )
""",

    f"{app_path}/main.py": """from fastapi import FastAPI, Depends, Query
from sqlalchemy.orm import Session
from fastapi_pagination import Page, add_pagination, paginate
from . import models, schemas, crud
from .database import SessionLocal, engine, Base

Base.metadata.create_all(bind=engine)

app = FastAPI(title="API Academia Crossfit")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/atletas/", response_model=schemas.AtletaResponse)
def criar_atleta(atleta: schemas.AtletaCreate, db: Session = Depends(get_db)):
    return crud.create_atleta(db=db, atleta=atleta)

@app.get("/atletas/", response_model=Page[schemas.AtletaResponse])
def listar_atletas(
    nome: str = Query(None, description="Filtrar por nome"),
    cpf: str = Query(None, description="Filtrar por CPF"),
    skip: int = 0,
    limit: int = 10,
    db: Session = Depends(get_db)
):
    atletas = crud.get_atletas(db=db, nome=nome, cpf=cpf, skip=skip, limit=limit)
    return paginate(atletas)

add_pagination(app)
""",

    f"{project_name}/.gitignore": """__pycache__/
*.pyc
*.db
.env
""",

    f"{project_name}/README.md": """# API Assíncrona para Academia de Crossfit
