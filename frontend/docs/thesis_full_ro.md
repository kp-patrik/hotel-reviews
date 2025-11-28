Titlu: Hotel Reviews — Analiză automată a recenziilor și sistem de sumarizare
Autor: Patrik Kiraly
Coordonator: Ș. I. dr. Gyorodi Cornelia, Facultatea de Inginerie Electrică și Tehnologia Informației
Specializarea: Calculatoare
An: 2025
Contact: patrik.kiraly17@gmail.com

---

Cuprins (generat automat la export DOCX)
- Rezumat
- 1. Introducere
- 2. Fundamente teoretice
- 3. Analiza cerințelor și specificațiilor
- 4. Proiectarea arhitecturii aplicației
- 5. Implementarea aplicației
- 6. Testare și validare
- 7. Concluzii, contribuții și direcții viitoare
- Anexe (cod, diagrame, DDL, instrucțiuni)

---

Rezumat
Acest proiect propune o aplicație web pentru colectarea, analiza și sumarizarea automată a recenziilor pentru hoteluri. Sistemul folosește tehnici NLP moderne pentru clasificare a sentimentului și sumarizare extractivă/abstractivă (dependențe și resurse disponibile). Documentația descrie arhitectura, implementarea, testarea și modul de rulare a aplicației.

Cuvinte-cheie: recenzii, NLP, sentiment analysis, summarization, FastAPI, React, BERT

---

1. Introducere
1.1 Context și motivație
Recenziile online sunt o resursă valoroasă dar dificil de parcurs în volum mare. Scopul aplicației este de a extrage semnale utile (sentiment, rezumat) din recenzii și de a le prezenta utilizatorului într-un mod rapid și accesibil.

1.2 Obiective
- Implementare backend (API REST), frontend (React) și model IA.
- Realizare sumarizare automată și clasificare a sentimentului.
- Evaluare experimentală a modelelor (baseline vs modern).

1.3 Structura lucrării
(vezi cuprins)

---

2. Fundamente teoretice
Prezintă conceptele esențiale necesare în proiect:
- NLP: tokenizare, embedding-uri, modelare secvențială.
- Clasificare sentiment: formulare, metrici (accuracy, precision, recall, F1).
- Sumarizare: extractivă vs abstractivă, metrici ROUGE.
- Probleme practice: lungimea textelor, bias, limbaj colocvial.

(Include explicații simple și tehnice, exemple, formule pentru metrici, referințe.)

---

3. Analiza cerințelor și specificațiilor
3.1 Actori
- Guest (vizualizare, generare sumar)
- Registered user (scriere recenzie, My Reviews)
- Admin (moderare)
- AI Service (componentă de inferență)

3.2 Cerințe funcționale
- CF-01: Vizualizare lista hoteluri și detalii.
- CF-02: Trimitere recenzie (anonym / authenticated).
- CF-03: Generare sumar pentru un hotel (asincron).
- CF-04: Analiză sentiment pentru recenzii (sinc/async).
- CF-05: Pagină My Reviews pentru utilizator autentificat.
- CF-06: Dashboard moderare (opțional).

3.3 Cerințe non-funcționale
- Scalabilitate (worker pentru sarcini IA).
- Securitate (JWT pentru autentificare).
- Performanță (cache, limitarea lungimii pentru inferență).
- GDPR: anonimizare, drept de ștergere.

3.4 Use Case + Component diagrams
Vezi diagramele incluse în directorul diagrams/:
- diagrams/usecase.png
- diagrams/components.png

Fiecare diagramă este urmată de o descriere în secțiunea Arhitectură.

---

4. Proiectarea arhitecturii aplicației
4.1 Tehnologii alese
- Backend: FastAPI (Python)
- Frontend: React + Vite
- DB: PostgreSQL
- AI: HuggingFace Transformers (BERT/BART), fallback TextRank
- Worker/Queue: Celery + Redis
- Containerizare: Docker Compose

4.2 Arhitectura logică (descriere)
- Frontend → API REST (autentificare JWT)
- API → DB, API → Worker (enqueue), Worker → AI Service (inferență)
- Sincron vs asincron:
  - Sumarizare: asincron (job enqueued)
  - Sentiment: poate fi sincron pentru texte scurte sau asincron pentru procesări în lot

4.3 Schema bazei de date
Tabele principale:
- users (id, username, email, password_hash, role, created_at)
- hotels (id, name, city, country, created_at)
- reviews (id, hotel_id, user_id, reviewer_name, review_text, rating, sentiment_label, sentiment_score, created_at)
- summaries (id, hotel_id, summary_text, method, created_at)

SQL DDL este disponibil în annex/db/schema.sql

4.4 Diagrame (vizuale)
- Use Case: diagrams/usecase.png — descrie actorii și acțiunile lor.
- Component: diagrams/components.png — prezintă Frontend, API, Worker, DB, Storage, AI.
- Sequence (generate summary): diagrams/sequence_summarize.png — fluxul de generare sumar.
- DB ER: diagrams/db_schema.png — relațiile entităților.

Pentru fiecare diagramă, se oferă o scurtă descriere în documentul final.

---

5. Implementarea aplicației
Prezentare pas-cu-pas a componentelor cheie, cu blocuri de cod esențiale (în engleză) și explicații accesibile.

5.1 Backend (exemple)
- Endpoint exemplu (POST /reviews): descriere, payload, verificări, enqueuing job.
  Fragment exemplar:
```python
@app.post("/reviews", response_model=schemas.ReviewOut, status_code=201)
def create_review(review_in: schemas.ReviewCreate, db: Session = Depends(models.get_db), current_user: schemas.UserOut | None = Depends(auth.get_current_user_optional)):
    user_id = current_user.id if current_user else None
    review = crud.create_review(db, review_in, user_id=user_id)
    services.enqueue_review_analysis(review.id, hotel_id=review.hotel_id)
    return review
```

5.2 Auth (JWT)
- Descriere: folosim dependency pentru a obține utilizatorul curent din header Authorization (Bearer token).
- Fragment (exemplu):
```python
def get_current_user(authorization: Optional[str] = Header(None)):
    if not authorization:
        raise HTTPException(status_code=401, detail="Missing Authorization")
    # decode token and return user object
```
Cod complet în Anexe.

5.3 Worker / Queue
- Folosim Celery + Redis. Task-urile în worker procesează sumarizare și analiză sentiment.
Exemplu de task:
```python
@celery.task(bind=True, acks_late=True)
def summarize_task(self, summary_id: int, hotel_id: int):
    # load reviews from DB, compute summary via ai_inference.summarize_reviews_texts
    # save summary via crud.save_summary_text
    pass
```

5.4 AI inference (esential)
- Sentiment: HF pipeline "sentiment-analysis" (BERT) sau fallback heuristics.
- Summarizer: HF summarizer (BART) sau fallback extractiv (nltk/TextRank).
Fragment (ai_inference.py):
```python
def analyze_sentiment_for_text(text: str) -> Tuple[str, float]:
    if sentiment_pipe:
        res = sentiment_pipe(text[:512])
        label = res[0].get("label", "NEUTRAL").upper()
        score = float(res[0].get("score", 0.0))
        return label, score
    # fallback heuristics...
```

5.5 Frontend (UX flows)
- Pagini: Home, Hotels list, Hotel detail (reviews + summary), My Reviews, Sentiment Analyzer.
- Componente esențiale: ReviewForm (submit), ReviewList, SummaryCard, AuthProvider.
- Exemplu request frontend:
```javascript
fetch('/reviews', { method: 'POST', headers: {...}, body: JSON.stringify(payload) })
```

5.6 Probleme întâmpinate & soluții
- CORS, token handling, async job visibility, rate limits, model cold-start.

---

6. Testare și validare
6.1 Metodologie
- Unit tests (pytest), integration tests, e2e (Cypress optional).
- Dataset split: train/val/test.

6.2 Metrici
- Sentiment: accuracy, precision, recall, F1.
- Summarization: ROUGE scores.

6.3 Rezultate (placeholder)
- Inserți tabelele experimentale când sunt disponibile.

6.4 Exemple vizuale (capturi)
- Hotel detail + summary: frontend/docs/images/03_hotel_detail.png
- Sentiment analyzer: frontend/docs/images/05_sentiment_analyzer.png
- My Reviews + footer: frontend/docs/images/09_my_reviews_footer.png
- Home mobile/desktop: frontend/docs/images/06_home_mobile.png, frontend/docs/images/01_home_desktop.png

---

7. Concluzii, contribuții, direcții viitoare
- Sumar realizări
- Contribuții originale
- Limitări și propuneri viitoare (multi-limbaj, abstractive summarization, model serving)

---

Anexe
A. Cod (complet) — folder annex/backend, annex/training etc. (incluse în ZIP)
B. DDL: annex/db/schema.sql
C. Diagrame .puml (folder diagrams/)
D. Instrucțiuni generare DOCX/PDF (README)
E. Capturi ecran (images/)

---

Footer / Notă copyright
Această lucrare este proprietatea autorului și nu poate fi copiată sau redistribuită fără permisiunea autorului. © 2025 Patrik Kiraly

```
