# Implementation Roadmap

Kolejność prac — każdy etap, każdy plik, co robi i dlaczego.

---

## Etap 0 — Konfiguracja projektu

### `pyproject.toml` (główny folder)
Definiuje projekt jako instalowalną paczkę Pythona. Bez tego nie można importować modułów z `src/` w innych plikach. Zawiera też konfigurację narzędzi: black, isort, flake8, pytest.

### `requirements.txt` (główny folder)
Lista wszystkich zależności pip. Używana przez Docker i CI do instalacji środowiska.

### `configs/config.yaml`
Główny plik konfiguracyjny Hydra. Łączy wszystkie pozostałe configi przez sekcję `defaults`. Zawiera nazwę i wersję projektu.

### `configs/kb.yaml`
Parametry generatora bazy wiedzy: lista 10 systemów z ID i nazwami, liczba problemów i procedur na system, ścieżka wyjściowa, model Ollama i jego parametry (temperatura, URL).

### `configs/ner.yaml`
Parametry treningu NER: bazowy model spaCy, lista encji do rozpoznawania (SYSTEM, ERROR_CODE, PERSON_NAME, PESEL, USER_LOGIN, DEPARTMENT, IP_ADDRESS, DEVICE_NAME), parametry treningu (epoki, batch, dropout, split), ścieżki danych, konfiguracja MLflow.

### `configs/retrieval.yaml`
Parametry pipeline RAG: model bi-encodera, batch size, konfiguracja ChromaDB (ścieżka, nazwa kolekcji), top_k dla BM25 i dense search, model rerankera, top_k rerankera, próg confidence (0.5) poniżej którego następuje eskalacja.

### `configs/logging.yaml`
Konfiguracja loggera loguru: poziom logowania, format wiadomości (czas, poziom, plik, linia, treść), rotacja pliku logów co 10MB, retencja 7 dni, kompresja zip.

### `.pre-commit-config.yaml` (główny folder)
Hooki uruchamiane przed każdym git commit: black (formatowanie), isort (sortowanie importów), flake8 (linting). Zapobiega commitowaniu nieformatowanego kodu.

### `.github/workflows/ci.yml`
GitHub Actions — uruchamia testy i linting przy każdym push i pull request. Instaluje zależności, odpala pytest i flake8.

### `Makefile` (główny folder)
Skróty do częstych komend. Przykład: `make generate-kb` zamiast `python src/data/kb_generator.py`. Ułatwia pracę i dokumentuje jak uruchamiać poszczególne etapy.

---

## Etap 1 — Generator bazy wiedzy

### `src/data/kb_generator.py`
Główny skrypt generujący syntetyczną bazę wiedzy. Wczytuje config z `configs/kb.yaml`, iteruje przez 10 systemów i dla każdego wysyła dwa prompty do Ollamy: jeden po 10 problemów JSON, drugi po 10 procedur JSON. Każdy problem ma: doc_id, system, ticket_type (incident/request), priority (1-5), error_codes, description (ciągły tekst), causes, related_procedures, tags. Każda procedura ma: proc_id, system, procedure_type (repair/configuration/maintenance/diagnostic), steps z unikalnymi placeholderami {{nazwa}}, słownik placeholders z opisem co każdy oznacza. Wynik zapisuje do `data/raw/problems.json` i `data/raw/procedures.json`. Loguje postęp przez loguru.

### `src/data/ollama_client.py`
Klient do komunikacji z Ollama REST API (localhost:11434). Enkapsuluje wywołania HTTP, obsługuje błędy, retry przy timeout, parsuje odpowiedź JSON. Używany przez kb_generator i ticket_generator.

### `pipelines/dvc.yaml` — stage: generate_kb
Definicja etapu DVC. Komenda: `python src/data/kb_generator.py`. Zależności: `configs/kb.yaml`, `src/data/kb_generator.py`. Outputy: `data/raw/problems.json`, `data/raw/procedures.json`.

---

## Etap 2 — Generator ticketów do treningu NER

### `src/data/ticket_generator.py`
Wczytuje `data/raw/problems.json`. Dla każdego z 100 problemów wysyła prompt do Ollamy z poleceniem wygenerowania 5 ticketów w różnym stylu (formalny, nieformalny, techniczny, lakoniczny, szczegółowy). Część ticketów zawiera dane osobowe (PERSON_NAME, PESEL, USER_LOGIN), część nie — mix realistyczny. Automatycznie tworzy etykiety NER bo wie z jakiego problemu pochodzi każdy ticket (zna SYSTEM i ERROR_CODE). Zapisuje do `data/tickets/synthetic_tickets.json` (500 ticketów z etykietami). Loguje postęp.

Format wyjściowy jednego ticketu:
```json
{
  "text": "Jan Kowalski PESEL 90010112345 zgłasza błąd DCM-CONN-TIMEOUT w PACS",
  "source_problem": "PACS-ERR-004",
  "entities": [
    {"text": "Jan Kowalski",     "label": "PERSON_NAME", "start": 0,  "end": 12},
    {"text": "90010112345",      "label": "PESEL",       "start": 19, "end": 30},
    {"text": "DCM-CONN-TIMEOUT", "label": "ERROR_CODE",  "start": 44, "end": 60},
    {"text": "PACS",             "label": "SYSTEM",      "start": 63, "end": 67}
  ]
}
```

### `pipelines/dvc.yaml` — stage: generate_tickets
Komenda: `python src/data/ticket_generator.py`. Zależności: `data/raw/problems.json`, `src/data/ticket_generator.py`. Outputy: `data/tickets/synthetic_tickets.json`.

---

## Etap 3 — Chunking i indeksowanie w ChromaDB

### `src/data/chunker.py`
Wczytuje `data/raw/problems.json`. Dla każdego dokumentu tworzy jeden chunk tekstowy przez sklejenie pól: `[title] + [description] + [error_codes]`. Flat chunking — jeden dokument = jeden chunk. Zapisuje chunki z metadanymi do `data/processed/chunks.json`. Loguje postęp.

### `src/embeddings/encoder.py`
Ładuje model bi-encodera (sentence-transformers/all-MiniLM-L6-v2). Udostępnia metodę `encode(texts)` która przyjmuje listę tekstów i zwraca macierz wektorów. Obsługuje batch processing (batch_size z configu). Używany przez indexer.

### `src/embeddings/indexer.py`
Wczytuje `data/processed/chunks.json`. Inicjalizuje kolekcję ChromaDB. Dla każdego chunku: generuje wektor przez encoder, zapisuje do ChromaDB razem z metadanymi (system, system_id, type, ticket_type, priority, error_codes, doc_id) i oryginalnym tekstem. Loguje postęp i liczbę zaindeksowanych dokumentów.

### `pipelines/dvc.yaml` — stage: chunk i stage: index
Dwa osobne stage'y. chunk zależy od problems.json, index zależy od chunks.json.

---

## Etap 4 — Trening NER

### `src/ner/trainer.py`
Wczytuje `data/tickets/synthetic_tickets.json`. Konwertuje format JSON na format spaCy (DocBin). Dzieli dane 80/20 train/eval. Inicjalizuje model spaCy na bazie `pl_core_news_sm`. Trenuje przez 30 epok z dropoutem 0.2. Po każdej epoce liczy metryki na zbiorze eval (precision, recall, F1 dla każdej encji). Loguje metryki do MLflow. Zapisuje najlepszy model (najwyższy F1) do `models/ner/`. Loguje postęp przez loguru.

### `src/ner/predictor.py`
Ładuje wytrenowany model NER z `models/ner/`. Udostępnia metodę `predict(text)` która zwraca listę wykrytych encji z labelami i pozycjami. Obsługuje maskowanie PESEL — zastępuje wykryte encje PESEL stringiem `[MASKED]` w tekście przed dalszym przetwarzaniem. Zwraca osobno: zamaskowany tekst + słownik encji pogrupowanych po typie.

### `pipelines/dvc.yaml` — stage: train_ner
Komenda: `python src/ner/trainer.py`. Zależności: `data/tickets/synthetic_tickets.json`, `configs/ner.yaml`. Outputy: `models/ner/`. Metryki: `metrics/ner_metrics.json`.

---

## Etap 5 — Pipeline wyszukiwania

### `src/retrieval/bm25_retriever.py`
Wczytuje wszystkie chunki z `data/processed/chunks.json`. Buduje indeks BM25 (rank-bm25) na tekstach chunków. Udostępnia metodę `retrieve(query, top_k, filters)` która tokenizuje zapytanie, filtruje dokumenty po metadanych (system, ticket_type), zwraca top_k dokumentów z score BM25.

### `src/retrieval/dense_retriever.py`
Inicjalizuje kolekcję ChromaDB. Używa encodera do wektoryzacji zapytania. Udostępnia metodę `retrieve(query, top_k, filters)` która wysyła zapytanie do ChromaDB z filtrem metadanych, zwraca top_k dokumentów z score podobieństwa.

### `src/retrieval/hybrid_retriever.py`
Łączy wyniki BM25 i dense search przez RRF (Reciprocal Rank Fusion). Wzór: `score = 1/(60 + rank_dense) + 1/(60 + rank_bm25)`. Przyjmuje wyniki z obu retrieverów, oblicza score RRF dla każdego dokumentu, zwraca posortowaną listę. Dokument wysoko u obu retrieverów wygrywa.

### `src/retrieval/reranker.py`
Ładuje model cross-encoder (cross-encoder/ms-marco-MiniLM-L-6-v2). Przyjmuje pary (ticket, dokument) i oblicza score podobieństwa dla każdej pary. Zwraca posortowane wyniki. Porównuje najwyższy score z progiem confidence (0.5 z configu) — jeśli poniżej progu flaguje jako eskalacja.

### `pipelines/dvc.yaml` — brak osobnego stage
Retrieval działa online (przy obsłudze ticketu), nie ma osobnego etapu DVC.

---

## Etap 6 — Integracja końcowa

### `src/pipeline/service_desk.py`
Główna klasa `ServiceDeskPipeline` która łączy wszystkie komponenty. Metoda `process_ticket(text)`:
1. NER — wyciąga encje, maskuje PESEL
2. Automatyczny priority — na podstawie systemu i słów kluczowych
3. Metadata filter — buduje filtr dla ChromaDB
4. Hybrid search — BM25 + dense + RRF
5. Reranker — wybiera najlepszy dokument, sprawdza próg
6. Eskalacja jeśli score < próg
7. Lookup procedury po ID z procedures.json
8. Buduje prompt dla LLM z ticketem + problemem + procedurą + encjami osobowymi
9. Wywołuje Ollama, zwraca odpowiedź z wypełnionymi placeholderami

### `src/pipeline/prompt_builder.py`
Buduje prompt dla Ollamy. Przyjmuje: tekst ticketu, JSON problemu, JSON procedury, słownik encji osobowych z NER. Wstawia dane osobowe w miejsce placeholderów jeśli są dostępne, w przeciwnym razie generuje ogólny prompt bez personalizacji.

### `src/pipeline/priority_scorer.py`
Logika automatycznego nadawania priorytetu (1-5). Reguły: krytyczne systemy (ICU Monitor, PACS) → wyższy priorytet, słowa kluczowe ("awaria", "wszyscy", "nie działa") → priorytet w górę, ("sporadycznie", "jeden użytkownik") → priorytet w dół, dziedziczenie z dopasowanego problemu z KB.

---

## Etap 7 — Testy

### `tests/test_kb_generator.py`
Testy generatora KB: poprawny format JSON wyjściowego, obecność wymaganych pól, walidacja error_codes, obecność placeholderów w procedurach.

### `tests/test_ner.py`
Testy modelu NER: rozpoznawanie SYSTEM, ERROR_CODE, PERSON_NAME, maskowanie PESEL, obsługa ticketu bez danych osobowych.

### `tests/test_retrieval.py`
Testy pipeline RAG: BM25 zwraca wyniki, dense zwraca wyniki, RRF poprawnie łączy rankingi, reranker zwraca score, eskalacja przy niskim score.

### `tests/test_pipeline.py`
Testy end-to-end: ticket z danymi osobowymi → spersonalizowana odpowiedź, ticket bez danych → ogólna odpowiedź, ticket poza bazą wiedzy → eskalacja.

---

## Etap 8 — Docker i CI

### `docker/Dockerfile`
Obraz Pythona 3.11. Kopiuje projekt, instaluje zależności z requirements.txt, instaluje projekt przez pip install -e. Nie zawiera Ollamy — Ollama działa na hoście.

### `docker/docker-compose.yml`
Uruchamia kontener aplikacji. Montuje foldery data/ i models/ jako wolumeny (dane poza kontenerem). Ustawia zmienną OLLAMA_BASE_URL wskazującą na host.

### `.github/workflows/ci.yml`
Na każdy push: checkout kodu, setup Python 3.11, instalacja zależności, uruchomienie flake8, uruchomienie pytest. Nie trenuje NER w CI (za długo) — sprawdza tylko testy jednostkowe z mockami.

---

## Kolejność implementacji

```
0. pyproject.toml + configs/ + Makefile
1. ollama_client.py
2. kb_generator.py  →  dvc stage: generate_kb
3. ticket_generator.py  →  dvc stage: generate_tickets
4. chunker.py  →  dvc stage: chunk
5. encoder.py + indexer.py  →  dvc stage: index
6. trainer.py + predictor.py  →  dvc stage: train_ner
7. bm25_retriever.py
8. dense_retriever.py
9. hybrid_retriever.py
10. reranker.py
11. priority_scorer.py
12. prompt_builder.py
13. service_desk.py
14. testy
15. Dockerfile + docker-compose.yml
16. ci.yml
```
