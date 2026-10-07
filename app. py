import os
import re
import json
import sqlite3
import unicodedata
from typing import List, Optional
from fastapi import FastAPI, UploadFile, File, Form, HTTPException
from fastapi.staticfiles import StaticFiles
from fastapi.responses import JSONResponse, FileResponse
from pydantic import BaseModel
import requests
from pypdf import PdfReader
import docx
from duckduckgo_search import DDGS

DATA_DIR = "data"
UPLOAD_DIR = os.path.join(DATA_DIR, "uploads")
DB_PATH = os.path.join(DATA_DIR, "personal_ai.db")

os.makedirs(UPLOAD_DIR, exist_ok=True)

app = FastAPI(title="Personal AI Backend")

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS directories (
            id TEXT PRIMARY KEY,
            name TEXT UNIQUE,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS documents (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            dir_id TEXT,
            filename TEXT,
            file_path TEXT,
            uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS doc_chunks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            doc_id INTEGER,
            dir_id TEXT,
            structural_tag TEXT,
            content TEXT,
            FOREIGN KEY (doc_id) REFERENCES documents (id) ON DELETE CASCADE
        )
    """)
    cur.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS doc_chunks_fts USING fts5(
            content,
            content_rowid='id'
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS conversations (
            id TEXT PRIMARY KEY,
            title TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            conv_id TEXT,
            role TEXT,
            content TEXT,
            tokens INTEGER DEFAULT 0,
            cost REAL DEFAULT 0.0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS settings (
            key TEXT PRIMARY KEY,
            value TEXT
        )
    """)

    default_dirs = [("vama", "Vama"), ("legi", "Legi"), ("imprumuturi", "Împrumuturi"), 
                    ("facturi", "Facturi"), ("spatiu_personal", "Spațiu personal")]
    for d_id, d_name in default_dirs:
        cur.execute("INSERT OR IGNORE INTO directories (id, name) VALUES (?, ?)", (d_id, d_name))
    
    conn.commit()
    conn.close()

init_db()

def strip_accents(text: str) -> str:
    return ''.join(c for c in unicodedata.normalize('NFD', text) if unicodedata.category(c) != 'Mn')

def legal_chunker(text: str) -> List[dict]:
    pattern = r'(?=(?:(?:\n|\A)\s*(?:Articolul|Art\.)\s*\d+|(?:Capitolul|CAPITOLUL)\s+[IVXLCDM\d]+|(?:Secțiunea|SECȚIUNEA)\s+\d+))'
    raw_splits = re.split(pattern, text)
    chunks = []
    
    for split in raw_splits:
        cleaned = split.strip()
        if not cleaned or len(cleaned) < 15:
            continue
        
        tag_match = re.match(r'^((?:Articolul|Art\.)\s*\d+|Capitolul\s+[IVXLCDM\d]+)', cleaned, re.IGNORECASE)
        tag = tag_match.group(1) if tag_match else "General"
        
        if len(cleaned) > 1200:
            sub_splits = re.split(r'(?=(?:\n\s*\(\d+\)|\n\s*[a-z]\)))', cleaned)
            current_buffer = ""
            for sub in sub_splits:
                if len(current_buffer) + len(sub) < 1200:
                    current_buffer += "\n" + sub
                else:
                    if current_buffer.strip():
                        chunks.append({"tag": tag, "content": current_buffer.strip()})
                    current_buffer = sub
            if current_buffer.strip():
                chunks.append({"tag": tag, "content": current_buffer.strip()})
        else:
            chunks.append({"tag": tag, "content": cleaned})
            
    return chunks

def retrieve_relevant_chunks(dir_id: str, query: str, limit: int = 4) -> List[str]:
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    
    terms = [re.sub(r'[^a-zA-Z0-9]', '', w) for w in query.split()]
    terms = [t for t in terms if len(t) > 2]
    
    if not terms:
        conn.close()
        return []

    fts_query = " OR ".join(f'"{t}"*' for t in terms)
    sql = """
        SELECT c.content FROM doc_chunks c
        JOIN doc_chunks_fts f ON c.id = f.rowid
        WHERE c.dir_id = ? AND doc_chunks_fts MATCH ?
        ORDER BY bm25(doc_chunks_fts) ASC
        LIMIT ?
    """
    try:
        cur.execute(sql, (dir_id, fts_query, limit))
        results = [row[0] for row in cur.fetchall()]
    except Exception:
        results = []
    finally:
        conn.close()
        
    return results

def search_web(query: str, max_results: int = 3) -> str:
    try:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=max_results))
            if not results:
                return ""
            summary = "\n".join([f"- {r.get('title')}: {r.get('body')}" for r in results])
            return f"Informații de pe Web:\n{summary}"
    except Exception:
        return ""

def query_llm(provider: str, model_name: str, messages: list, api_key: str):
    if provider == "openai":
        url = "https://api.openai.com/v1/chat/completions"
        headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }
        payload = {
            "model": model_name or "gpt-4o-mini",
            "messages": messages,
            "temperature": 0.1
        }
        res = requests.post(url, headers=headers, json=payload, timeout=25)
        if res.status_code != 200:
            raise HTTPException(status_code=res.status_code, detail=f"OpenAI Error: {res.text}")
        data = res.json()
        answer = data["choices"][0]["message"]["content"]
        tokens = data.get("usage", {}).get("total_tokens", 0)
        return answer, tokens, 0.00015 * (tokens / 1000)

    elif provider == "gemini":
        model = model_name or "gemini-1.5-flash"
        url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={api_key}"
        
        contents = []
        for m in messages:
            role = "user" if m["role"] in ["user", "system"] else "model"
            contents.append({"role": role, "parts": [{"text": m["content"]}]})
            
        payload = {
            "contents": contents,
            "generationConfig": {"temperature": 0.1}
        }
        res = requests.post(url, json=payload, timeout=25)
        if res.status_code != 200:
            raise HTTPException(status_code=res.status_code, detail=f"Gemini Error: {res.text}")
        data = res.json()
        try:
            answer = data["candidates"][0]["content"]["parts"][0]["text"]
            tokens = data.get("usageMetadata", {}).get("totalTokenCount", 0)
            return answer, tokens, 0.000075 * (tokens / 1000)
        except (KeyError, IndexError):
            raise HTTPException(status_code=500, detail="Răspuns invalid primit de la Gemini.")
    else:
        raise HTTPException(status_code=400, detail="Furnizor necunoscut.")

@app.get("/api/directories")
def get_directories():
    conn = sqlite3.connect
