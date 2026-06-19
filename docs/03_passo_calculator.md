# Passo 2 – Modifica `src/calculator.py`

**Prerequisiti:** Passo 1 (`rules.py`) completato  
**Copre:** NC-1, NC-2, NC-3, NC-4, NC-6  
**Difficoltà:** ⭐⭐⭐ Alta – logica di calcolo nuova e regime transitorio

---

## Perché questo file va modificato

`calculator.py` esegue tutti i calcoli di esenzione. Deve essere reso **data-consapevole**:  
la data di sostenimento della richiesta determina quale set di massimali e plafond usare.  
Inoltre occorre implementare:

- La **riduzione progressiva** per le trasferte estere con più di 5 giorni (Sezione 4).
- Il **limite mensile di 12 giornate** per la categoria `lavoro_agile` (Sezione 3).
- Un parametro aggiuntivo nella firma di `calcola()` per ricevere le giornate di lavoro agile già rimborsate nel mese.

---

## Contenuto attuale del file

```python
"""Calcolo della quota esente e della quota imponibile di una richiesta."""

from src import rules


def massimale_teorico(richiesta):
    """Massimale di esenzione applicabile alla richiesta, in base alla categoria."""
    categoria = richiesta["categoria"]
    if categoria in rules.CATEGORIE_A_GIORNATE:
        return round(rules.MASSIMALI_GIORNALIERI[categoria] * richiesta["giorni"], 2)
    if categoria == "chilometrico":
        return round(rules.MASSIMALE_KM * richiesta["km"], 2)
    if categoria == "alloggio":
        return round(rules.MASSIMALE_NOTTE * richiesta["notti"], 2)
    raise ValueError(f"categoria non gestita: {categoria}")


def calcola(richiesta, esente_gia_riconosciuta):
    """Restituisce (quota_esente, quota_imponibile, dettaglio).

    `esente_gia_riconosciuta` è la quota esente già riconosciuta al dipendente
    nel mese della richiesta, ai fini del plafond mensile.
    """
    importo = richiesta["importo"]
    teorico = massimale_teorico(richiesta)
    esente_teorica = min(importo, teorico)
    capienza = max(rules.PLAFOND_MENSILE - esente_gia_riconosciuta, 0.0)
    esente = round(min(esente_teorica, capienza), 2)
    imponibile = round(importo - esente, 2)
    dettaglio = {
        "massimale_teorico": teorico,
        "esente_teorica": round(esente_teorica, 2),
        "capienza_plafond": round(capienza, 2),
    }
    return esente, imponibile, dettaglio
```

---

## Nuovo contenuto completo del file

Sostituisci l'**intero contenuto** di `src/calculator.py` con il seguente:

```python
"""Calcolo della quota esente e della quota imponibile di una richiesta."""

from datetime import date

from src import rules


# ---------------------------------------------------------------------------
# Funzioni di supporto per il regime transitorio
# ---------------------------------------------------------------------------

def _is_2026(richiesta):
    """True se la data di sostenimento è dal 01/01/2026 (nuova disciplina)."""
    return date.fromisoformat(richiesta["data"]) >= rules.DATA_DECORRENZA_2026


def _massimali_per_data(richiesta):
    """Restituisce (massimali_giornalieri, massimale_km, massimale_notte, plafond_mensile)
    in base alla data di sostenimento della richiesta.

    Regime transitorio (Sezione 7, Circ. 18/2026):
      - data <= 31/12/2025 → valori Circ. 41/2024
      - data >= 01/01/2026 → valori Circ. 18/2026
    """
    if _is_2026(richiesta):
        return (
            rules.MASSIMALI_GIORNALIERI_2026,
            rules.MASSIMALE_KM_2026,
            rules.MASSIMALE_NOTTE_2026,
            rules.PLAFOND_MENSILE_2026,
        )
    return (
        rules.MASSIMALI_GIORNALIERI_2025,
        rules.MASSIMALE_KM_2025,
        rules.MASSIMALE_NOTTE_2025,
        rules.PLAFOND_MENSILE_2025,
    )


# ---------------------------------------------------------------------------
# Riduzione progressiva trasferte estere >5 giorni (Sezione 4, Circ. 18/2026)
# ---------------------------------------------------------------------------

def _massimale_estero_progressivo(giorni):
    """Massimale teorico per trasferte estere dal 2026 con riduzione progressiva.

    Formulazione dalla norma:
      G1 = min(G, 5)            quota piena:        G1 × 85,00
      G2 = min(max(G−5, 0), 5)  quota ridotta 10%:  G2 × 76,50
      G3 = max(G−10, 0)         quota ridotta 20%:  G3 × 68,00
    """
    g1 = min(giorni, 5)
    g2 = min(max(giorni - 5, 0), 5)
    g3 = max(giorni - 10, 0)
    totale = (
        g1 * 85.00
        + g2 * rules.MASSIMALE_ESTERO_RIDUZIONE_10
        + g3 * rules.MASSIMALE_ESTERO_RIDUZIONE_20
    )
    return round(totale, 2)


# ---------------------------------------------------------------------------
# API pubblica
# ---------------------------------------------------------------------------

def massimale_teorico(richiesta):
    """Massimale di esenzione applicabile alla richiesta.

    Tiene conto di:
    - regime transitorio (data 2025 vs 2026)
    - riduzione progressiva per trasferte estere >5 giorni (solo 2026)

    ATTENZIONE: per la categoria lavoro_agile, il campo richiesta["giorni"]
    deve già contenere le giornate ammesse (non le giornate richieste).
    Il troncamento al limite mensile viene applicato in calcola().
    """
    categoria = richiesta["categoria"]
    massimali_g, massimale_km, massimale_notte, _ = _massimali_per_data(richiesta)

    # Trasferta estero 2026: riduzione progressiva oltre 5 giorni
    if categoria == "trasferta_estero" and _is_2026(richiesta):
        return _massimale_estero_progressivo(richiesta["giorni"])

    # Categorie a giornate (trasferta_italia, pasto, lavoro_agile)
    if categoria in rules.CATEGORIE_A_GIORNATE:
        return round(massimali_g[categoria] * richiesta["giorni"], 2)

    if categoria == "chilometrico":
        return round(massimale_km * richiesta["km"], 2)

    if categoria == "alloggio":
        return round(massimale_notte * richiesta["notti"], 2)

    raise ValueError(f"categoria non gestita: {categoria}")


def calcola(richiesta, esente_gia_riconosciuta, giorni_lavoro_agile_nel_mese=0):
    """Calcola quota esente e quota imponibile. Restituisce (esente, imponibile, dettaglio).

    Parametri:
      richiesta                    – dizionario della richiesta
      esente_gia_riconosciuta      – quota esente già attribuita al dipendente
                                     nel mese (per il plafond mensile)
      giorni_lavoro_agile_nel_mese – giornate di lavoro agile già rimborsate
                                     nel mese (usato solo per la categoria
                                     lavoro_agile, per il cap di 12 gg/mese)
    """
    importo = richiesta["importo"]
    _, _, _, plafond = _massimali_per_data(richiesta)

    # Per lavoro_agile: ridurre i giorni alle giornate effettivamente ammesse
    # prima di calcolare il massimale (Sezione 3, Circ. 18/2026)
    richiesta_per_calcolo = richiesta
    if richiesta["categoria"] == "lavoro_agile":
        giorni_richiesti = richiesta["giorni"]
        giorni_ammessi = min(
            giorni_richiesti,
            max(rules.MAX_GIORNI_LAVORO_AGILE_MENSILI - giorni_lavoro_agile_nel_mese, 0),
        )
        # Copia parziale con i giorni ammessi (non modifica la richiesta originale)
        richiesta_per_calcolo = {**richiesta, "giorni": giorni_ammessi}

    teorico = massimale_teorico(richiesta_per_calcolo)
    esente_teorica = min(importo, teorico)
    capienza = max(plafond - esente_gia_riconosciuta, 0.0)
    esente = round(min(esente_teorica, capienza), 2)
    imponibile = round(importo - esente, 2)
    dettaglio = {
        "massimale_teorico": teorico,
        "esente_teorica": round(esente_teorica, 2),
        "capienza_plafond": round(capienza, 2),
    }
    return esente, imponibile, dettaglio
```

---

## Spiegazione delle modifiche principali

### 1. `_is_2026(richiesta)` (nuova funzione privata)
Converte `richiesta["data"]` in un oggetto `date` e lo confronta con `DATA_DECORRENZA_2026`.  
Restituisce `True` se la richiesta è soggetta alla nuova disciplina.

### 2. `_massimali_per_data(richiesta)` (nuova funzione privata)
Restituisce la tupla corretta di massimali e plafond in base alla data.  
Questo sostituisce il riferimento diretto alle costanti globali.

### 3. `_massimale_estero_progressivo(giorni)` (nuova funzione privata)
Implementa la formula della Sezione 4 della circolare:
- Giorni 1–5: 85,00 € pieno
- Giorni 6–10: 76,50 € (-10%)
- Giorni 11+: 68,00 € (-20%)

### 4. `massimale_teorico()` – modifiche
- Riceve i massimali tramite `_massimali_per_data()` invece delle costanti globali.
- Per `trasferta_estero` + data 2026 chiama `_massimale_estero_progressivo()`.

### 5. `calcola()` – modifiche
- Nuovo parametro opzionale `giorni_lavoro_agile_nel_mese` (default 0).
- Se la categoria è `lavoro_agile`, calcola le giornate ammesse come `min(giorni_richiesti, 12 − già_rimborsate)`.
- Il plafond viene preso da `_massimali_per_data()` invece che da `rules.PLAFOND_MENSILE`.

---

## Come verificare che la modifica sia corretta

### Verifica rapida (terminale)

```bash
python -c "
from src import calculator

# Caso Sezione 6.2 (norma): trasferta estero 6 gg, 2026, importo 500
r = {'dipendente': 'X', 'data': '2026-03-01', 'categoria': 'trasferta_estero',
     'importo': 500.00, 'giorni': 6, 'km': None, 'notti': None}
e, i, d = calculator.calcola(r, 0)
print('Caso 6.2 – esente:', e, '(atteso: 500.00), imponibile:', i, '(atteso: 0.00)')
print('  massimale_teorico:', d['massimale_teorico'], '(atteso: 501.50)')

# Caso Sezione 6.4 (norma): lavoro agile 15 gg, già 0, importo 52.50
r2 = {'dipendente': 'X', 'data': '2026-04-01', 'categoria': 'lavoro_agile',
      'importo': 52.50, 'giorni': 15, 'km': None, 'notti': None}
e2, i2, d2 = calculator.calcola(r2, 0, giorni_lavoro_agile_nel_mese=0)
print('Caso 6.4 – esente:', e2, '(atteso: 42.00), imponibile:', i2, '(atteso: 10.50)')

# Regime transitorio: pasto 2025 usa massimale 8.00, non 10.00
r3 = {'dipendente': 'X', 'data': '2025-12-18', 'categoria': 'pasto',
      'importo': 24.00, 'giorni': 3, 'km': None, 'notti': None}
e3, _, d3 = calculator.calcola(r3, 0)
print('Caso 6.6 – massimale_teorico:', d3['massimale_teorico'], '(atteso: 24.00, ovvero 3×8.00)')
"
```

### Caso norma 6.1 (Sezione 6.1)

```bash
python -c "
from src import calculator
# Pasto 5 giorni, importo 50, esente_gia=1350, plafond 1400 → capienza 50 → esente 50
r = {'dipendente': 'X', 'data': '2026-05-01', 'categoria': 'pasto',
     'importo': 50.00, 'giorni': 5, 'km': None, 'notti': None}
e, i, d = calculator.calcola(r, 1350.00)
print('Caso 6.1 – esente:', e, '(atteso: 50.00), imponibile:', i, '(atteso: 0.00)')
"
```

### Caso norma 6.3 (Sezione 6.3 – trasferta estero esattamente 5 gg: no riduzione)

```bash
python -c "
from src import calculator
r = {'dipendente': 'X', 'data': '2026-03-01', 'categoria': 'trasferta_estero',
     'importo': 500.00, 'giorni': 5, 'km': None, 'notti': None}
_, _, d = calculator.calcola(r, 0)
print('Caso 6.3 – massimale_teorico:', d['massimale_teorico'], '(atteso: 425.00)')
"
```

---

## Attenzione: modifica della firma di `calcola()`

La funzione `calcola()` ora accetta un terzo parametro opzionale:

```python
# PRIMA
calculator.calcola(richiesta, esente_gia_riconosciuta)

# DOPO
calculator.calcola(richiesta, esente_gia_riconosciuta, giorni_lavoro_agile_nel_mese=0)
```

Il parametro è **opzionale** (default = 0), quindi le chiamate esistenti in `app.py` e nei test  
continueranno a funzionare senza modifiche immediate. Solo il Passo 5 (`app.py`) lo passerà  
esplicitamente per le richieste di lavoro agile.
