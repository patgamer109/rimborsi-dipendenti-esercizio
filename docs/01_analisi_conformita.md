# Analisi di conformità – Circolare MEF n. 18/2026

**Documento:** Analisi tecnica del codice rispetto alle specifiche  
**Fonte normativa:** `docs/ITA_circolare_mef_18_2026.pdf`  
**Data analisi:** 2026-06-19  
**Analista:** Senior Analyst

---

## Sintesi esecutiva

Il software è attualmente conforme alla **Circolare MEF n. 41/2024** (anno 2025).  
La **Circolare MEF n. 18/2026**, in vigore dal 01/01/2026, introduce **6 novità** che richiedono modifiche al codice.  
Nessuna delle 6 novità è stata implementata. Il sistema applica massimali e regole obsolete a tutte le richieste, indipendentemente dalla data di sostenimento.

---

## Mappa delle non-conformità

| # | Novità normativa (Circ. 18/2026) | File coinvolto | Stato |
|---|----------------------------------|----------------|-------|
| NC-1 | Massimali aggiornati dal 01/01/2026 | `rules.py`, `calculator.py` | ❌ Non implementato |
| NC-2 | Plafond mensile elevato a 1.400 €/mese dal 01/01/2026 | `rules.py`, `calculator.py`, `app.py` | ❌ Non implementato |
| NC-3 | Nuova categoria "indennità lavoro agile" (max 12 gg/mese) | `rules.py`, `calculator.py`, `storage.py`, `validator.py`, `app.py`, template, JS | ❌ Non implementato |
| NC-4 | Riduzione progressiva trasferte estere > 5 giorni | `calculator.py` | ❌ Non implementato |
| NC-5 | Incompatibilità lavoro agile / trasferta | `validator.py`, `app.py` | ❌ Non implementato |
| NC-6 | Regime transitorio (regole 2025 per date ≤ 31/12/2025) | `calculator.py`, `validator.py`, `rules.py` | ❌ Non implementato |

**Bug aggiuntivo trovato durante l'analisi:**

| # | Descrizione | File | Stato |
|---|-------------|------|-------|
| BUG-1 | La data di sostenimento non viene verificata come non futura (Sez. 5 Circ. 18/2026) | `validator.py` | ❌ Mancante |

---

## Dettaglio non-conformità

### NC-1 – Massimali aggiornati

**Norma (Sezione 2):** Dal 01/01/2026 i massimali giornalieri cambiano.

| Categoria | Valore attuale nel codice | Valore corretto 2026 |
|-----------|--------------------------|---------------------|
| `trasferta_italia` | 46,48 €/giorno | **50,00 €/giorno** |
| `trasferta_estero` | 77,47 €/giorno | **85,00 €/giorno** |
| `pasto` | 8,00 €/giorno | **10,00 €/giorno** |
| `chilometrico` | 0,42 €/km | **0,45 €/km** |
| `alloggio` | 150,00 €/notte | **170,00 €/notte** |

**Impatto:** Tutte le richieste del 2026 ricevono rimborsi sottostimati.

---

### NC-2 – Plafond mensile

**Norma (Sezione 2):** Il plafond mensile passa da 1.200 € a **1.400 €** dal 01/01/2026.

**Codice attuale (`rules.py`):**
```python
PLAFOND_MENSILE = 1200.00
```

**Impatto:** Il sistema respinge o riduce quote che la norma 2026 dovrebbe riconoscere come esenti.

---

### NC-3 – Categoria "lavoro agile"

**Norma (Sezione 3):** Nuova categoria `lavoro_agile`:
- Massimale: **3,50 €/giorno**
- Limite mensile: **12 giornate** (le giornate eccedenti sono integralmente imponibili)
- Si cumula al plafond mensile come le altre categorie

**Codice attuale:** La categoria non esiste né in `rules.CATEGORIE` né in `CATEGORIE_A_GIORNATE`. Il calcolo in `calculator.py` non gestisce né il massimale né il cap di 12 giornate mensili. La funzione in `storage.py` per contare le giornate già rimborsate nel mese non esiste.

---

### NC-4 – Riduzione progressiva trasferte estere

**Norma (Sezione 4):** Per trasferte estere con data inizio dal 01/01/2026 e durata > 5 giorni:
- Giorni 1–5: massimale pieno (**85,00 €**)
- Giorni 6–10: massimale ridotto del 10% (**76,50 €**)
- Giorni 11 in poi: massimale ridotto del 20% (**68,00 €**)

**Formula:**
```
G1 = min(G, 5)           → G1 × 85,00
G2 = min(max(G−5, 0), 5) → G2 × 76,50
G3 = max(G−10, 0)        → G3 × 68,00
massimale_teorico = G1×85 + G2×76,50 + G3×68
```

**Codice attuale (`calculator.py`):**
```python
if categoria in rules.CATEGORIE_A_GIORNATE:
    return round(rules.MASSIMALI_GIORNALIERI[categoria] * richiesta["giorni"], 2)
```
Applica sempre il massimale pieno, senza alcuna riduzione progressiva.

**Esempio (dalla norma):** Trasferta di 12 giorni → massimale corretto 943,50 €; il codice calcolerebbe erroneamente 77,47 × 12 = **929,64 €** (2025) o 85 × 12 = **1.020,00 €** (2026, senza riduzione).

---

### NC-5 – Incompatibilità lavoro agile / trasferta

**Norma (Sezione 5):** Una richiesta di `lavoro_agile` è respinta se almeno una delle giornate a cui si riferisce coincide con una giornata di trasferta valida dello stesso dipendente (e viceversa). La motivazione è `"incompatibilità lavoro agile / trasferta"`. La regola vale solo per date dal 01/01/2026.

**Codice attuale (`validator.py`):** La funzione `valida()` non accetta né riceve le richieste già presenti. Non può fare alcun confronto con lo storico.

---

### NC-6 – Regime transitorio

**Norma (Sezione 7):** Le richieste con data di sostenimento ≤ 31/12/2025 usano i massimali 41/2024 e il plafond di 1.200 €. La categoria `lavoro_agile` non è ammessa per date 2025. La riduzione progressiva e l'incompatibilità si applicano solo dal 01/01/2026.

**Codice attuale:** Non esiste alcuna logica data-dipendente. Il codice applica una sola serie di costanti (quelle 2025, già obsolete).

---

### BUG-1 – Data futura non bloccata

**Norma (Sezione 5):** "data di sostenimento presente e **non successiva alla data di presentazione**".

**Codice attuale (`validator.py`):** La data viene solo verificata come parsabile ISO, non confrontata con `date.today()`.

---

## Ordine di intervento consigliato

Le modifiche devono essere eseguite **nell'ordine seguente** per evitare dipendenze non soddisfatte:

```
Passo 1 → src/rules.py        (aggiungere costanti 2025/2026 e nuova categoria)
Passo 2 → src/calculator.py   (calcolo date-aware, riduzione progressiva, lavoro agile)
Passo 3 → src/storage.py      (funzione conteggio giornate lavoro agile nel mese)
Passo 4 → src/validator.py    (incompatibilità, regime transitorio, data futura)
Passo 5 → src/app.py          (cablaggio del nuovo validator e del nuovo calculator)
Passo 6 → src/static/app.js   (aggiunta lavoro_agile alla mappa campi)
           src/templates/      (normativa aggiornata, riepilogo date-aware)
Passo 7 → tests/              (aggiornare test obsoleti, aggiungere test per le nuove regole)
```

I singoli passi sono descritti nei documenti `02_passo_*.md` – `08_passo_*.md`.

---

## Riepilogo impatto per file

| File | Tipo modifica | Criticità |
|------|--------------|-----------|
| `src/rules.py` | Aggiunta costanti + nuova categoria | Alta |
| `src/calculator.py` | Logica date-aware + 2 nuovi algoritmi | Alta |
| `src/storage.py` | +1 funzione | Bassa |
| `src/validator.py` | +2 controlli, firma modificata | Alta |
| `src/app.py` | Cablaggio, +1 helper, fix riepilogo | Media |
| `src/static/app.js` | +1 riga nella mappa campi | Bassa |
| `src/templates/normativa.html` | Tabella aggiornata + sezioni nuove | Bassa |
| `src/templates/riepilogo.html` | Plafond per-riga | Bassa |
| `tests/test_calculator.py` | Aggiornamento + nuovi test | Media |
| `tests/test_validator.py` | Aggiornamento + nuovi test | Media |
| `tests/test_app.py` | Aggiornamento + nuovi test | Media |
