# API Set Up Guide


## First Reorganise Project Structure
```
root (EXERCISES.JSON/)
│
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── database.py
│   │   │   └── config.py
│   │   ├── models/
│   │   │   ├── exercise_model.py
│   │   │   └── __init__.py
│   │   ├── schemas/
│   │   │   ├── exercise_schema.py
│   │   │   └── __init__.py
│   │   ├── services/
│   │   │   ├── exercise_service.py
│   │   │   └── __init__.py
│   │   ├── routes/
│   │   │   ├── exercise_routes.py
│   │   │   └── __init__.py
│   │   └── main.py
│   │
│   ├── tests/
│   │   ├── test_models.py
│   │   ├── test_routes.py
│   │   └── test_services.py
│   │
│   ├── requirements.txt
│   └── README.md
│
├── data/
│   ├── exercises/ # all the JSON exercises and images go here
|   |
|   | 
├── db/
│   ├── database.schema.table 
│   ├── seed.sql # A SQL file to seed the database on deploy
│
└── README.md
```

## Set Up
1. After completing the above reorganisation set up your Python `virtualenv` > `cd` into the `backend/` folder and install the project dependencies and move onto the Tasks next.

2. Create the appropriate files referenced in the tasks below, copy and paste the stub code, and run it. See Setup and Installation Instructions below if you get stuck.

## Tasks

### 1. Database Configuration (config/database.py)
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os

# Database connection configuration - when you set up the Postgres DB set the url and connection string in a `.env` file
DATABASE_URL = os.getenv(
    'DATABASE_URL', 
    'postgresql://username:password@localhost/exercise_db'
)

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 2. Exercise Model (models/exercise_model.py)
```python
from sqlalchemy import Column, Integer, String, JSON, DateTime
from sqlalchemy.sql import func
from ..config.database import Base

class Exercise(Base):
    __tablename__ = "exercises"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    category = Column(String)
    metadata = Column(JSON)
    video_path = Column(String, nullable=True)
    image_paths = Column(JSON, nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### 3. Exercise Schema (schemas/exercise_schema.py)
```python
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime

class ExerciseBase(BaseModel):
    name: str
    category: Optional[str] = None
    metadata: dict
    video_path: Optional[str] = None
    image_paths: Optional[List[str]] = None

class ExerciseCreate(ExerciseBase):
    pass

class Exercise(ExerciseBase):
    id: int
    created_at: datetime
    updated_at: Optional[datetime] = None

    class Config:
        orm_mode = True
```

### 4. Exercise Service (services/exercise_service.py)
```python
from sqlalchemy.orm import Session
from ..models.exercise_model import Exercise as ExerciseModel
from ..schemas.exercise_schema import ExerciseCreate

class ExerciseService:
    @staticmethod
    def create_exercise(db: Session, exercise: ExerciseCreate):
        db_exercise = ExerciseModel(**exercise.dict())
        db.add(db_exercise)
        db.commit()
        db.refresh(db_exercise)
        return db_exercise

    @staticmethod
    def get_exercise_by_id(db: Session, exercise_id: int):
        return db.query(ExerciseModel).filter(ExerciseModel.id == exercise_id).first()

    @staticmethod
    def get_exercises_by_category(db: Session, category: str):
        return db.query(ExerciseModel).filter(ExerciseModel.category == category).all()

    @staticmethod
    def update_exercise(db: Session, exercise_id: int, exercise_update: ExerciseCreate):
        db_exercise = db.query(ExerciseModel).filter(ExerciseModel.id == exercise_id).first()
        if db_exercise:
            for key, value in exercise_update.dict(exclude_unset=True).items():
                setattr(db_exercise, key, value)
            db.commit()
            db.refresh(db_exercise)
        return db_exercise

    @staticmethod
    def delete_exercise(db: Session, exercise_id: int):
        db_exercise = db.query(ExerciseModel).filter(ExerciseModel.id == exercise_id).first()
        if db_exercise:
            db.delete(db_exercise)
            db.commit()
        return db_exercise
```

### 5. Exercise Routes (routes/exercise_routes.py)
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from ..config.database import get_db
from ..services.exercise_service import ExerciseService
from ..schemas.exercise_schema import ExerciseCreate, Exercise

router = APIRouter(prefix="/exercises", tags=["exercises"])

@router.post("/", response_model=Exercise)
def create_exercise(exercise: ExerciseCreate, db: Session = Depends(get_db)):
    return ExerciseService.create_exercise(db=db, exercise=exercise)

@router.get("/{exercise_id}", response_model=Exercise)
def read_exercise(exercise_id: int, db: Session = Depends(get_db)):
    db_exercise = ExerciseService.get_exercise_by_id(db, exercise_id=exercise_id)
    if db_exercise is None:
        raise HTTPException(status_code=404, detail="Exercise not found")
    return db_exercise

@router.get("/category/{category}", response_model=list[Exercise])
def read_exercises_by_category(category: str, db: Session = Depends(get_db)):
    return ExerciseService.get_exercises_by_category(db, category=category)

@router.put("/{exercise_id}", response_model=Exercise)
def update_exercise(exercise_id: int, exercise: ExerciseCreate, db: Session = Depends(get_db)):
    updated_exercise = ExerciseService.update_exercise(db, exercise_id=exercise_id, exercise_update=exercise)
    if updated_exercise is None:
        raise HTTPException(status_code=404, detail="Exercise not found")
    return updated_exercise

@router.delete("/{exercise_id}", response_model=Exercise)
def delete_exercise(exercise_id: int, db: Session = Depends(get_db)):
    deleted_exercise = ExerciseService.delete_exercise(db, exercise_id=exercise_id)
    if deleted_exercise is None:
        raise HTTPException(status_code=404, detail="Exercise not found")
    return deleted_exercise
```

### 6. Main Application (main.py)
```python
from fastapi import FastAPI
from .config.database import engine
from .models import exercise_model
from .routes import exercise_routes

# Create database tables
exercise_model.Base.metadata.create_all(bind=engine)

app = FastAPI(title="Exercise Recognition API")

# Include routers
app.include_router(exercise_routes.router)

@app.get("/")
def read_root():
    return {"message": "Welcome to Exercise Recognition API"}
```

### 7. Requirements (requirements.txt)
```
fastapi==0.95.1
uvicorn==0.22.0
sqlalchemy==1.4.46
psycopg2-binary==2.9.6
pydantic==1.10.7
python-dotenv==1.0.0
```

## Setup and Installation Instructions
1. Create a virtual environment
2. Install dependencies: `pip install -r requirements.txt`
3. Set up PostgreSQL database
4. Configure DATABASE_URL environment variable
5. Run migrations if using Alembic
6. Start the server: `uvicorn src.main:app --reload`

## Next Steps (DJ TODO)
- Implement data ingestion script for JSON files
- Add authentication
- Create test coverage
- Set up CI/CD pipeline with Digital Ocean
```
Tasks

1. Database Setup
- Create PostgreSQL database
- Configure database connection
- Set up environment variables

2. Project Structure Implementation
- Create the directory structure as outlined
- Implement each Python file with the provided code
- Set up virtual environment
- Install dependencies

3. Data Ingestion
- Create a script to read JSON files
- Parse exercise metadata
- Insert data into PostgreSQL database
- Handle file path management for videos/images

4. Testing
- Write unit tests for models
- Create integration tests for API routes
- Test database operations

5. Documentation
- Add docstrings to functions
- Create a README with setup instructions
- Document API endpoints
