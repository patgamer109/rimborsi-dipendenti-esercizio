# Passo 4 – Modifica `src/validator.py`

**Prerequisiti:** Passo 1 (`rules.py`) completato  
**Copre:** NC-3 (validazione lavoro agile 2025), NC-5 (incompatibilità), NC-6 (regime transitorio), BUG-1 (data futura)  
**Difficoltà:** ⭐⭐⭐ Alta – nuova logica di incompatibilità e firma modificata

---

## Perché questo file va modificato

La Sezione 5 della Circolare 18/2026 introduce tre nuove regole di validazione:

1. **BUG-1 – Data futura non ammessa:** la data di sostenimento non può essere successiva  
   alla data odierna di presentazione.
2. **NC-6 – Regime transitorio per `lavoro_agile`:** la categoria `lavoro_agile` non  
   è ammessa per date anteriori al 01/01/2026 (motivazione: "categoria non riconosciuta").
3. **NC-5 – Incompatibilità lavoro agile / trasferta:** una richiesta di `lavoro_agile`  
   è respinta se anche una sola delle giornate coperte coincide con una giornata di  
   trasferta valida dello stesso dipendente, e viceversa.

Il punto 3 richiede che `valida()` riceva l'elenco delle richieste già presenti in storage.  
La firma della funzione viene modificata aggiungendo un parametro opzionale.

---

## Come funziona il controllo di incompatibilità

Le richieste di tipo `trasferta` e `lavoro_agile` coprono un **intervallo di date**:
- `data` è la data di inizio
- `giorni` è il numero di giorni consecutivi

Per trovare l'insieme di date coperte da una richiesta si usa:
```
{data_inizio + k giorni  per k in 0, 1, ..., giorni-1}
```

Due richieste sono incompatibili se il loro insieme di date si sovrappone **e**  
almeno una delle date sovrapposte è dal 01/01/2026.

---

## Contenuto attuale del file

```python
"""Regole di validazione delle richieste di rimborso."""

from datetime import date

from src import rules


def valida(richiesta):
    """Restituisce (True, "") se la richiesta è valida, altrimenti (False, motivazione)."""
    if not richiesta.get("dipendente"):
        return False, "dipendente mancante"

    categoria = richiesta.get("categoria")
    if categoria not in rules.CATEGORIE:
        return False, "categoria non riconosciuta"

    importo = richiesta.get("importo")
    if importo is None or importo <= 0:
        return False, "importo non positivo"

    try:
        date.fromisoformat(richiesta.get("data") or "")
    except ValueError:
        return False, "data mancante o non valida"

    if categoria in rules.CATEGORIE_A_GIORNATE:
        giorni = richiesta.get("giorni")
        if not giorni or giorni <= 0:
            return False, "numero di giornate non valido"

    if categoria == "chilometrico":
        km = richiesta.get("km")
        if not km or km <= 0:
            return False, "numero di chilometri non valido"

    if categoria == "alloggio":
        notti = richiesta.get("notti")
        if not notti or notti <= 0:
            return False, "numero di notti non valido"

    return True, ""
```

---

## Nuovo contenuto completo del file

Sostituisci l'**intero contenuto** di `src/validator.py` con il seguente:

```python
"""Regole di validazione delle richieste di rimborso."""

from datetime import date, timedelta

from src import rules


# ---------------------------------------------------------------------------
# Funzione privata: verifica incompatibilità lavoro agile / trasferta
# ---------------------------------------------------------------------------

def _date_coperte(richiesta):
    """Restituisce il set di date (oggetti date) coperte dalla richiesta.

    Per categorie a giornate la richiesta copre giorni consecutivi a partire
    da data; per le altre categorie copre solo la data stessa.
    """
    inizio = date.fromisoformat(richiesta["data"])
    giorni = richiesta.get("giorni") or 1
    return {inizio + timedelta(days=k) for k in range(giorni)}


def _controlla_incompatibilita(richiesta, richieste_esistenti):
    """Verifica l'incompatibilità tra lavoro agile e trasferta (Sezione 5).

    Restituisce (True, "") se nessuna incompatibilità, altrimenti
    (False, "incompatibilità lavoro agile / trasferta").

    La regola si applica solo alle date dal 01/01/2026.
    Sono rilevanti solo le richieste con stato 'valida'.
    """
    categoria = richiesta.get("categoria", "")
    dipendente = richiesta.get("dipendente", "")

    # Solo le combinazioni lavoro_agile ↔ trasferta sono rilevanti
    e_lavoro_agile = categoria == "lavoro_agile"
    e_trasferta = categoria in rules.CATEGORIE_TRASFERTA
    if not e_lavoro_agile and not e_trasferta:
        return True, ""

    # Date coperte dalla nuova richiesta, limitate al 2026
    date_nuova = _date_coperte(richiesta)
    date_nuova_2026 = {d for d in date_nuova if d >= rules.DATA_DECORRENZA_2026}
    if not date_nuova_2026:
        # Tutte le giornate sono in regime 2025: incompatibilità non applicata
        return True, ""

    for r in richieste_esistenti:
        if r.get("stato") != "valida":
            continue
        if r.get("dipendente") != dipendente:
            continue

        cat_r = r.get("categoria", "")

        # Controllo incrociato: lavoro_agile vs trasferta (in entrambe le direzioni)
        conflitto = (
            (e_lavoro_agile and cat_r in rules.CATEGORIE_TRASFERTA)
            or (e_trasferta and cat_r == "lavoro_agile")
        )
        if not conflitto:
            continue

        # Calcola le date coperte dall'altra richiesta, limitate al 2026
        date_r = _date_coperte(r)
        date_r_2026 = {d for d in date_r if d >= rules.DATA_DECORRENZA_2026}

        # Se c'è almeno un giorno in comune → incompatibilità
        if date_nuova_2026 & date_r_2026:
            return False, "incompatibilità lavoro agile / trasferta"

    return True, ""


# ---------------------------------------------------------------------------
# API pubblica
# ---------------------------------------------------------------------------

def valida(richiesta, richieste_esistenti=None):
    """Restituisce (True, "") se la richiesta supera tutti i controlli,
    altrimenti (False, motivazione_del_respingimento).

    Parametri:
      richiesta           – dizionario della richiesta da validare
      richieste_esistenti – lista delle richieste già registrate in storage
                            (necessaria per il controllo di incompatibilità);
                            se None il controllo di incompatibilità è saltato
    """
    if not richiesta.get("dipendente"):
        return False, "dipendente mancante"

    categoria = richiesta.get("categoria")
    if categoria not in rules.CATEGORIE:
        return False, "categoria non riconosciuta"

    importo = richiesta.get("importo")
    if importo is None or importo <= 0:
        return False, "importo non positivo"

    # Validazione della data
    try:
        data_sos = date.fromisoformat(richiesta.get("data") or "")
    except ValueError:
        return False, "data mancante o non valida"

    # BUG-1: la data di sostenimento non può essere nel futuro (Sezione 5)
    if data_sos > date.today():
        return False, "data futura non ammessa"

    # Regime transitorio: lavoro_agile non ammesso per date 2025 (Sezione 7)
    if categoria == "lavoro_agile" and data_sos < rules.DATA_DECORRENZA_2026:
        return False, "categoria non riconosciuta"

    if categoria in rules.CATEGORIE_A_GIORNATE:
        giorni = richiesta.get("giorni")
        if not giorni or giorni <= 0:
            return False, "numero di giornate non valido"

    if categoria == "chilometrico":
        km = richiesta.get("km")
        if not km or km <= 0:
            return False, "numero di chilometri non valido"

    if categoria == "alloggio":
        notti = richiesta.get("notti")
        if not notti or notti <= 0:
            return False, "numero di notti non valido"

    # NC-5: incompatibilità lavoro agile / trasferta
    if richieste_esistenti is not None:
        ok, motivazione = _controlla_incompatibilita(richiesta, richieste_esistenti)
        if not ok:
            return False, motivazione

    return True, ""
```

---

## Spiegazione delle modifiche

### 1. `_date_coperte(richiesta)` (nuova funzione privata)
Calcola l'insieme (`set`) di tutti i giorni coperti da una richiesta.  
Per una trasferta di 3 giorni iniziata il 2026-03-06:
```
{2026-03-06, 2026-03-07, 2026-03-08}
```

### 2. `_controlla_incompatibilita(richiesta, richieste_esistenti)` (nuova funzione privata)
- Verifica solo se la categoria è `lavoro_agile` o `trasferta_*`.
- Filtra le date della nuova richiesta a quelle dal 01/01/2026 (regime transitorio).
- Per ogni richiesta valida dello stesso dipendente che sia dell'altro tipo,  
  calcola l'intersezione delle date 2026. Se non è vuota → respinge.

### 3. `valida()` – modifiche
- Aggiunto parametro opzionale `richieste_esistenti=None`.
- Aggiunto controllo **data futura** (BUG-1).
- Aggiunto controllo **lavoro_agile ante-2026** (regime transitorio NC-6).
- Aggiunta chiamata a `_controlla_incompatibilita()` solo se `richieste_esistenti` è fornito.

---

## Come verificare che la modifica sia corretta

### Test base – richiesta valida (non deve cambiare nulla)

```bash
python -c "
from src import validator
r = {'dipendente': 'Mario', 'data': '2026-01-10', 'categoria': 'pasto',
     'importo': 20.0, 'giorni': 2, 'km': None, 'notti': None}
print(validator.valida(r, []))
# Atteso: (True, '')
"
```

### Test regime transitorio – lavoro agile con data 2025

```bash
python -c "
from src import validator
r = {'dipendente': 'Mario', 'data': '2025-12-01', 'categoria': 'lavoro_agile',
     'importo': 10.5, 'giorni': 3, 'km': None, 'notti': None}
print(validator.valida(r, []))
# Atteso: (False, 'categoria non riconosciuta')
"
```

### Test incompatibilità – Caso 6.5 della norma

```bash
python -c "
from src import validator

# Richiesta esistente: trasferta nazionale 02/03/2026-06/03/2026
trasferta = {'id': 1, 'dipendente': 'Mario', 'data': '2026-03-02',
             'categoria': 'trasferta_italia', 'stato': 'valida',
             'importo': 200.0, 'giorni': 5, 'quota_esente': 200.0, 'quota_imponibile': 0.0}

# Nuova richiesta: lavoro agile 06/03/2026-08/03/2026 (3 giorni)
nuova = {'dipendente': 'Mario', 'data': '2026-03-06', 'categoria': 'lavoro_agile',
         'importo': 10.5, 'giorni': 3, 'km': None, 'notti': None}

print(validator.valida(nuova, [trasferta]))
# Atteso: (False, 'incompatibilità lavoro agile / trasferta')
# (il 06/03/2026 è compreso in entrambe le richieste)
"
```

### Test incompatibilità – nessun overlap → valida

```bash
python -c "
from src import validator

trasferta = {'id': 1, 'dipendente': 'Mario', 'data': '2026-03-02',
             'categoria': 'trasferta_italia', 'stato': 'valida',
             'importo': 200.0, 'giorni': 5, 'quota_esente': 200.0, 'quota_imponibile': 0.0}

# Lavoro agile parte il 07/03 (trasferta finisce il 06/03 incluso)
nuova = {'dipendente': 'Mario', 'data': '2026-03-07', 'categoria': 'lavoro_agile',
         'importo': 10.5, 'giorni': 2, 'km': None, 'notti': None}

print(validator.valida(nuova, [trasferta]))
# Atteso: (True, '')
"
```

---

## Impatto sulle chiamate esistenti

La firma `valida(richiesta)` è ancora valida: `richieste_esistenti` ha default `None`.  
Le chiamate nei test esistenti non necessitano modifiche immediate.  
Solo `app.py` (passo 5) passerà il secondo argomento in produzione.
