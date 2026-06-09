# Hybrid Search Architecture

## Pełny Flow

```
Ticket (tekst wejściowy)
         │
         ▼
┌─────────────────┐
│   NER Model     │  wyciąga: system, error_code, severity
└─────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│         Metadata Filter                 │
│  WHERE system = "PACS"                  │
│  (twarde zawężenie — eliminuje złe sys) │
└─────────────────────────────────────────┘
         │
         ▼
    20 dokumentów PACS
         │
         ├─────────────────────┐
         ▼                     ▼
┌──────────────┐     ┌──────────────────┐
│  Bi-encoder  │     │      BM25        │
│  (semantyka, │     │  (dokładne słowa │
│   znaczenie) │     │   i kody błędów) │
└──────────────┘     └──────────────────┘
         │                     │
    Top-10 dense          Top-10 BM25
         │                     │
         └──────────┬──────────┘
                    ▼
         ┌─────────────────┐
         │   RRF Fusion    │  łączy rankingi po pozycji
         └─────────────────┘
                    │
                    ▼
         ┌─────────────────┐
         │    Reranker     │  cross-encoder, wybiera Top-3
         └─────────────────┘
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
  score > 0.5            score < 0.5
         │                     │
         ▼                     ▼
  Ollama LLM            ESKALACJA
  → odpowiedź           → człowiek
```

## Co robi każdy element

| Element | Typ filtrowania | Na czym działa | Kiedy |
|---------|----------------|----------------|-------|
| Metadata filter | twarde zero-jedynkowe | pola strukturalne (system, type) | przed wyszukiwaniem |
| Bi-encoder | ranking semantyczny | znaczenie tekstu | podczas wyszukiwania |
| BM25 | ranking słów kluczowych | dokładne stringi w tekście | podczas wyszukiwania |
| RRF Fusion | łączenie rankingów | pozycje z obu wyników | po wyszukiwaniu |
| Reranker | końcowy ranking | para (ticket, dokument) | finalna selekcja |

## Kiedy co pomaga

| Sytuacja | Metadata | Bi-encoder | BM25 |
|----------|----------|------------|------|
| Ticket ma nazwę systemu | TAK | nie | nie |
| Ticket ma kod błędu (DCM-CONN-TIMEOUT) | nie | słabo | TAK |
| Ticket opisuje problem słowami | nie | TAK | częściowo |
| NER się pomyli | zawodzi | działa | działa |

## Chunking (parent-child)

```
Dokument KB
      │
      ├── Child chunk (128 tokenów) ← używany do retrieval
      │   [symptomy + tagi + error_codes]
      │
      └── Parent chunk (512 tokenów) ← zwracany do LLM
          [pełny dokument z rozwiązaniem]
```

## RRF — jak liczy score

```
score_RRF(doc) = 1/(k + rank_dense) + 1/(k + rank_bm25)
k = 60 (stała wygładzająca)

Przykład:
PACS-ERR-004: 1/(60+1) + 1/(60+1) = 0.0328  ← wysoki (top u obu)
PACS-ERR-009: 1/(60+2) + 1/(60+5) = 0.0315
PACS-ERR-011: 1/(60+3) + 1/(60+8) = 0.0303
```
