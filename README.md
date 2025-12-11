from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.orm import Session
from db import SessionLocal, init_db
from models import TaxonRecord
from import_data import import_csv_if_empty
from typing import List, Optional
from pydantic import BaseModel
import os

app = FastAPI(title="Material → Genus abundance API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# serve frontend static files
app.mount("/static", StaticFiles(directory=os.path.join(os.path.dirname(__file__), "..", "static")), name="static")
app.mount("/web", StaticFiles(directory=os.path.join(os.path.dirname(__file__), "..", "web")), name="web")

@app.on_event("startup")
def startup():
    init_db()
    # import CSV into DB if empty
    csv_path = os.getenv("CSV_PATH", "/app/data/AAA_tidy_taxa.csv")
    import_csv_if_empty(csv_path)

class TaxonOut(BaseModel):
    genus: str
    taxon: str
    material: str
    depth: int
    abundance: float

@app.get("/api/materials", response_model=List[str])
def list_materials():
    db: Session = SessionLocal()
    mats = db.query(TaxonRecord.material).distinct().all()
    return [m[0] for m in mats]

@app.get("/api/material/{material_name}", response_model=List[TaxonOut])
def get_material(material_name: str, depth: int = 1, genus: Optional[str] = None, limit: int = 1000):
    db: Session = SessionLocal()
    q = db.query(TaxonRecord).filter(TaxonRecord.material.ilike(material_name))
    q = q.filter(TaxonRecord.depth == depth)
    if genus:
        q = q.filter(TaxonRecord.genus.ilike(f"%{genus}%"))
    q = q.order_by(TaxonRecord.abundance.desc()).limit(limit)
    results = q.all()
    if not results:
        raise HTTPException(status_code=404, detail="No data for material/depth/genus")
    return results

@app.get("/", include_in_schema=False)
def index():
    return FileResponse(os.path.join(os.path.dirname(__file__), "..", "web", "index.html"))
