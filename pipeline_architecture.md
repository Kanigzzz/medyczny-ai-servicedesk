# Pipeline Architecture

## Co wektoryzujemy i co nie

```
ChromaDB (wektoryzowane):
└── 100 problemów  ← tu szukamy przez bi-encoder + BM25

data/raw/procedures.json (plain JSON):
└── 100 procedur   ← tu pobieramy po ID, zero wyszukiwania
```

Procedury NIE są wektoryzowane — zawsze pobieramy je po ID
z pola related_procedures w dokumencie problemu.

---

## Format dokumentu — Problem (ChromaDB)

```json
{
  "doc_id": "PACS-ERR-004",
  "system": "PACS",
  "system_id": "SYS-01",
  "type": "problem",
  "ticket_type": "incident",
  "title": "Brak ładowania obrazów DICOM po aktualizacji systemu",
  "priority": 1,
  "error_codes": ["PACS-ERR-004", "DCM-CONN-TIMEOUT"],
  "description": "Po aktualizacji systemu PACS użytkownicy nie mogą otwierać badań. Przeglądarka obrazów zawiesza się i po minucie wyświetla błąd DCM-CONN-TIMEOUT. Problem dotyczy wszystkich stacji roboczych w oddziale radiologii.",
  "causes": [
    "Nieprawidłowa konfiguracja portu DICOM po aktualizacji",
    "Usługa wada-service nie uruchomiła się automatycznie"
  ],
  "related_procedures": ["PACS-PROC-002", "PACS-PROC-007"],
  "tags": ["network", "imaging", "post-update", "service-failure"],
  "language": "pl"
}
```

Decyzje projektowe:
- `description` → ciągły tekst naturalny (nie lista) — lepiej matchuje z ticketem dla bi-encodera i BM25
- `ticket_type` → incident lub request (patrz niżej)
- `priority` → liczba 1-5, nadawana AUTOMATYCZNIE przez system, nigdy przez użytkownika
- `solution_steps` → usunięte — kroki są w procedure JSON
- `tags` → ogólne kategorie (network, imaging itd.), przyszły metadata filter gdy użytkownik wybierze kategorię w formularzu

---

## ticket_type — incident vs request

```
incident  = coś się zepsuło po stronie systemu/oprogramowania
            użytkownik nie może pracować
            przykład: "PACS nie ładuje obrazów, błąd DCM-CONN-TIMEOUT"

request   = błędna konfiguracja po stronie użytkownika
            system działa poprawnie, użytkownik źle go używa/skonfigurował
            przykład: "nie mogę się zalogować bo mam złe uprawnienia"
```

`ticket_type` trafia do metadanych ChromaDB — pozwala filtrować:
```
WHERE system = "PACS" AND ticket_type = "incident"
```

---

## priority — nadawany automatycznie

Użytkownik NIGDY nie określa priorytetu. System nadaje go automatycznie.
Dotyczy tylko `ticket_type: incident` — requesty nie mają priorytetu krytyczności.

Skala:
```
1 → najwyższy priorytet — system niedostępny, wpływ na pacjentów
2 → poważna degradacja, część funkcji niedostępna
3 → problem ale workaround istnieje
4 → sporadyczny, nie blokuje pracy
5 → najniższy priorytet — kosmetyczny
```

Na podstawie czego system nadaje priority:
```
1. Systemu którego dotyczy ticket:
   ICU Monitor, PACS → domyślnie wyższy priorytet (1-2)
   Billing, Scheduling → domyślnie niższy (3-4)

2. Słów kluczowych w description:
   "nie działa", "awaria", "wszyscy użytkownicy" → priorytet w górę
   "czasem", "sporadycznie", "jeden użytkownik"  → priorytet w dół

3. Dopasowanego problemu z KB:
   problem w KB ma już przypisany priority → system go dziedziczy
```

---

## Format dokumentu — Procedura (plain JSON)

```json
{
  "proc_id": "PACS-PROC-002",
  "system": "PACS",
  "system_id": "SYS-01",
  "type": "procedure",
  "procedure_type": "repair",
  "title": "Restart usługi wada-service",
  "when_to_use": "Gdy obrazy DICOM nie ładują się lub timeout połączenia",
  "prerequisites": ["Dostęp SSH do serwera", "Uprawnienia sudo"],
  "steps": [
    "Zaloguj się do serwera przez SSH jako {{admin_username}}",
    "Sprawdź status usługi: systemctl status wada-service",
    "Zrestartuj usługę: systemctl restart wada-service",
    "Zweryfikuj status: systemctl status wada-service",
    "Sprawdź logi: journalctl -u wada-service -n 50",
    "Potwierdź z użytkownikiem {{user_fullname}} czy problem ustąpił"
  ],
  "placeholders": {
    "admin_username": "nazwa konta administratora z ticketu",
    "user_fullname":  "imię i nazwisko zgłaszającego z ticketu"
  },
  "tags": ["network", "service-management", "post-update"]
}
```

### procedure_type — rodzaje procedur

```
repair          → naprawa zepsutego systemu (incident)
configuration   → poprawna konfiguracja po stronie użytkownika (request)
maintenance     → rutynowe czynności administracyjne
diagnostic      → jak zdiagnozować problem, zebrać logi
```

Powiązanie z ticket_type:
```
ticket incident → szukaj procedur repair + diagnostic
ticket request  → szukaj procedur configuration
```

### Placeholdery {{nazwa}}

Wypełniane przez LLM automatycznie na podstawie danych wyciągniętych przez NER z ticketu.

WAŻNE: każda procedura ma UNIKALNE placeholdery dopasowane do swojego kontekstu.
Generator KB musi tworzyć różne zestawy dla każdej procedury:

```
PACS-PROC-002 (restart usługi)
  {{admin_username}}, {{user_fullname}}

EHR-PROC-005 (reset hasła)
  {{user_fullname}}, {{user_login}}, {{department}}

LIS-PROC-003 (dodanie uprawnienia)
  {{user_fullname}}, {{lab_unit}}, {{permission_level}}

HIS-PROC-007 (konfiguracja drukarki)
  {{printer_ip}}, {{ward_name}}, {{user_fullname}}
```

Dane osobowe są OPCJONALNE — użytkownik nie musi podawać imienia/nazwiska:
```
Ticket z danymi:
  NER wyciąga PERSON_NAME → {{user_fullname}} = "Jan Kowalski"
  LLM generuje spersonalizowane rozwiązanie

Ticket bez danych:
  NER nie znajduje PERSON_NAME → {{user_fullname}} = brak
  LLM generuje ogólne rozwiązanie bez personalizacji
```

---

## NER — trzy role

```
Rola 1 — Retrieval:
  wyciąga: SYSTEM, ERROR_CODE
  → idą do metadata filter + BM25

Rola 2 — Placeholder filling:
  wyciąga: PERSON_NAME, USER_LOGIN, DEPARTMENT, IP_ADDRESS, DEVICE_NAME
  → wypełniają {{placeholdery}} w procedurze przez LLM

Rola 3 — Anonimizacja:
  wyciąga: PESEL
  → zastępowany [MASKED] przed przekazaniem do LLM i logów
```

Maskowanie PESEL:
```
Wejście:  "Jan Kowalski, PESEL 90010112345, nie może zalogować się do EHR"
Po NER:   "Jan Kowalski, PESEL [MASKED], nie może zalogować się do EHR"
→ zamaskowana wersja idzie do logów i ChromaDB query
→ PERSON_NAME trafia osobno do placeholdera {{user_fullname}}
```

---

## Co idzie do ChromaDB (tekst do wektoryzacji)

```
[title] + [description] + [error_codes]
```

Przykład:
```
"Brak ładowania obrazów DICOM po aktualizacji systemu.
 Po aktualizacji systemu PACS użytkownicy nie mogą otwierać badań.
 Przeglądarka obrazów zawiesza się i po minucie wyświetla błąd DCM-CONN-TIMEOUT.
 Problem dotyczy wszystkich stacji roboczych w oddziale radiologii.
 PACS-ERR-004 DCM-CONN-TIMEOUT"
```

Ten tekst → bi-encoder → wektor [768 liczb] → ChromaDB

Metadane zapisywane osobno w ChromaDB:
```python
metadata = {
    "system":       "PACS",
    "system_id":    "SYS-01",
    "type":         "problem",
    "ticket_type":  "incident",
    "priority":     "1",
    "error_codes":  "PACS-ERR-004,DCM-CONN-TIMEOUT",
    "doc_id":       "PACS-ERR-004"
}
```

---

## Pełny Flow Online (obsługa ticketu)

```
Ticket użytkownika
"Jan Kowalski PESEL 90010112345 - po aktualizacji PACS
 nie mogę otworzyć badań, błąd DCM-CONN-TIMEOUT"
         │
         ▼
┌──────────────────────────────────────────┐
│               NER Model                  │
│                                          │
│  retrieval:    SYSTEM=PACS               │
│                ERROR_CODE=DCM-CONN-TIMEOUT│
│                                          │
│  placeholdery: PERSON_NAME=Jan Kowalski  │
│                                          │
│  maskowanie:   PESEL → [MASKED]          │
└──────────────────────────────────────────┘
         │
         ├─────────────────────────────────────────┐
         ▼                                         ▼
Encje retrieval                          Encje osobowe
(SYSTEM, ERROR_CODE)                     (PERSON_NAME itd.)
         │                                         │
         ▼                                         ▼
┌──────────────────────────┐        trzymane osobno do
│    Automatyczny priority  │        wypełnienia placeholderów
│  system + słowa kluczowe │
│  → priority = 1          │
│  (tylko incident)        │
└──────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│         Metadata Filter              │
│  WHERE system = "PACS"               │
│  AND type = "problem"                │
│  Zawęża 100 problemów → 10 PACS      │
└──────────────────────────────────────┘
         │
         ▼
    10 problemów PACS
         │
         ├──────────────────────────┐
         ▼                          ▼
┌──────────────────┐    ┌───────────────────────┐
│   Bi-encoder     │    │         BM25           │
│  ticket → wektor │    │  szuka słów z ticketu  │
│  porównuje z     │    │  w description prob.   │
│  wektorami PACS  │    │  (timeout, aktualizac) │
│  Top-10 semantic │    │  Top-10 keyword        │
└──────────────────┘    └───────────────────────┘
         │                          │
         └────────────┬─────────────┘
                      ▼
         ┌────────────────────────┐
         │       RRF Fusion       │
         │  score = 1/(60+rank_dense)
         │        + 1/(60+rank_bm25)
         │  wygrywa ten co jest   │
         │  wysoko u OBU          │
         └────────────────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │        Reranker        │
         │    (cross-encoder)     │
         │  sprawdza parę:        │
         │  (ticket, problem)     │
         │  → score podobieństwa  │
         └────────────────────────┘
                      │
         ┌────────────┴─────────────┐
         ▼                          ▼
   score > 0.5                score < 0.5
         │                          │
         ▼                          ▼
Pobierz pełny JSON             ESKALACJA
problemu PACS-ERR-004          → człowiek
         │
         ▼
related_procedures: ["PACS-PROC-002"]
         │
         ▼
Pobierz PACS-PROC-002
z procedures.json (po ID, bez wyszukiwania)
         │
         ▼
┌──────────────────────────────────────┐
│             Ollama LLM               │
│  Kontekst:                           │
│  - treść ticketu (z [MASKED] PESEL)  │
│  - problem JSON (description, causes,│
│    ticket_type, priority)            │
│  - procedura JSON (steps z {{...}})  │
│  - encje osobowe z NER               │
│                                      │
│  LLM wypełnia {{placeholdery}}       │
│  danymi z NER lub generuje ogólne    │
│  rozwiązanie jeśli brak danych       │
└──────────────────────────────────────┘
         │
         ▼
Odpowiedź dla użytkownika
```

---

## Offline — Generowanie danych

### Krok 1 — Generowanie KB (Ollama)

```
configs/kb.yaml (10 systemów, 10 prob, 10 proc)
         │
         ▼
kb_generator.py
  dla każdego systemu:
    prompt → Ollama → 10 problemów JSON (incident + request mix)
    prompt → Ollama → 10 procedur JSON (repair + configuration mix)
    każda procedura dostaje unikalne placeholdery dla swojego kontekstu
         │
         ▼
data/raw/problems.json    (100 dokumentów)
data/raw/procedures.json  (100 dokumentów)
```

### Krok 2 — Generowanie ticketów do treningu NER (Ollama)

```
data/raw/problems.json
         │
         ▼
ticket_generator.py
  dla każdego problemu (100x):
    prompt → Ollama → 5 ticketów w różnym stylu
    część ticketów zawiera dane osobowe (PERSON_NAME, PESEL, USER_LOGIN)
    część ticketów bez danych osobowych (ogólne zgłoszenie)
    automatyczne labelowanie NER (wiemy z jakiego problemu pochodzi)
         │
         ▼
data/tickets/synthetic_tickets.json  (500 ticketów z etykietami)
```

Format ticketu z etykietami NER:
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

Encje NER:
```
retrieval:     SYSTEM, ERROR_CODE
placeholdery:  PERSON_NAME, USER_LOGIN, DEPARTMENT, IP_ADDRESS, DEVICE_NAME
maskowanie:    PESEL → zastępowany [MASKED] przed LLM i logami
```

### Krok 3 — Chunking i indeksowanie

```
data/raw/problems.json
         │
         ▼
chunker.py
  child chunk (128 tokenów) ← do retrieval (wektoryzacja)
  parent chunk (512 tokenów) ← zwracany do LLM jako kontekst
         │
         ▼
indexer.py
  bi-encoder → wektor z [title + description + error_codes]
  zapis do ChromaDB z metadanymi
         │
         ▼
data/chromadb/   (gotowa baza wektorowa)
```

### Krok 4 — Trening NER

```
data/tickets/synthetic_tickets.json (500 ticketów)
         │
         ▼
trainer.py (spaCy)
  train/eval split 80/20
  30 epok
  logi metryk do MLflow
         │
         ▼
models/ner/   (gotowy model NER)
```

---

## Podsumowanie danych

| Co | Format | Gdzie | Ile |
|----|--------|-------|-----|
| Problemy (surowe) | JSON | data/raw/problems.json | 100 |
| Procedury (surowe) | JSON | data/raw/procedures.json | 100 |
| Problemy (wektory) | ChromaDB | data/chromadb/ | 100 |
| Procedury (lookup) | JSON | data/raw/procedures.json | 100 |
| Tickety NER | JSON | data/tickets/ | 500 |
| Model NER | spaCy | models/ner/ | 1 |
