# Passo 3 – Modifica `src/storage.py`

**Prerequisiti:** Passo 1 (`rules.py`) completato  
**Copre:** NC-3 (parte relativa al conteggio giornate lavoro agile)  
**Difficoltà:** ⭐ Bassa – aggiunta di una sola funzione

---

## Perché questo file va modificato

La Sezione 3 della Circolare 18/2026 stabilisce che le giornate di lavoro agile esenti  
non possono superare **12 per mese di calendario** per dipendente.  
Il calcolatore (passo 2) ha bisogno di sapere quante giornate sono già state rimborsate  
nel mese corrente per determinare le "giornate ammesse".

`storage.py` gestisce già una funzione analoga per il plafond mensile (`esente_riconosciuta_nel_mese`);  
va aggiunta una funzione equivalente che conta le **giornate di lavoro agile** già approvate.

---

## Contenuto attuale del file

```python
"""Persistenza delle richieste su file JSON."""

import json
from pathlib import Path

PERCORSO_DATI = Path(__file__).resolve().parent.parent / "data" / "richieste.json"


def carica():
    ...

def salva(richieste):
    ...

def prossimo_id(richieste):
    return max((r["id"] for r in richieste), default=0) + 1

def mese(richiesta):
    """Mese di riferimento di una richiesta, nel formato AAAA-MM."""
    return richiesta["data"][:7]

def esente_riconosciuta_nel_mese(richieste, dipendente, mese_riferimento):
    """Somma delle quote esenti delle richieste valide del dipendente nel mese."""
    totale = sum(
        r["quota_esente"]
        for r in richieste
        if r["dipendente"] == dipendente
        and r["stato"] == "valida"
        and mese(r) == mese_riferimento
    )
    return round(totale, 2)
```

---

## Modifica da eseguire

Aggiungi la seguente funzione **in fondo al file**, dopo `esente_riconosciuta_nel_mese`:

```python
def giornate_lavoro_agile_nel_mese(richieste, dipendente, mese_riferimento):
    """Numero totale di giornate di lavoro agile già rimborsate al dipendente nel mese.

    Conta solo le richieste con stato 'valida' e categoria 'lavoro_agile'.
    Usato dal calcolatore per applicare il cap di 12 giornate/mese (Sezione 3,
    Circolare MEF n. 18/2026).
    """
    return sum(
        r.get("giorni", 0) or 0
        for r in richieste
        if r["dipendente"] == dipendente
        and r["stato"] == "valida"
        and r.get("categoria") == "lavoro_agile"
        and mese(r) == mese_riferimento
    )
```

### Dove inserirla nel file

Apri `src/storage.py` e aggiungi il blocco dopo l'ultima riga esistente della funzione  
`esente_riconosciuta_nel_mese`. Il file finirà così:

```python
def esente_riconosciuta_nel_mese(richieste, dipendente, mese_riferimento):
    """Somma delle quote esenti delle richieste valide del dipendente nel mese."""
    totale = sum(
        r["quota_esente"]
        for r in richieste
        if r["dipendente"] == dipendente
        and r["stato"] == "valida"
        and mese(r) == mese_riferimento
    )
    return round(totale, 2)


def giornate_lavoro_agile_nel_mese(richieste, dipendente, mese_riferimento):
    """Numero totale di giornate di lavoro agile già rimborsate al dipendente nel mese.

    Conta solo le richieste con stato 'valida' e categoria 'lavoro_agile'.
    Usato dal calcolatore per applicare il cap di 12 giornate/mese (Sezione 3,
    Circolare MEF n. 18/2026).
    """
    return sum(
        r.get("giorni", 0) or 0
        for r in richieste
        if r["dipendente"] == dipendente
        and r["stato"] == "valida"
        and r.get("categoria") == "lavoro_agile"
        and mese(r) == mese_riferimento
    )
```

---

## Come verificare che la modifica sia corretta

```bash
python -c "
from src import storage

# Dati di test: 2 richieste valide di lavoro agile nello stesso mese
richieste = [
    {'id': 1, 'dipendente': 'Mario', 'data': '2026-04-05', 'categoria': 'lavoro_agile',
     'stato': 'valida', 'giorni': 8, 'quota_esente': 28.00, 'quota_imponibile': 0.0},
    {'id': 2, 'dipendente': 'Mario', 'data': '2026-04-20', 'categoria': 'pasto',
     'stato': 'valida', 'giorni': 2, 'quota_esente': 20.00, 'quota_imponibile': 0.0},
    {'id': 3, 'dipendente': 'Mario', 'data': '2026-04-22', 'categoria': 'lavoro_agile',
     'stato': 'respinta', 'giorni': 3, 'quota_esente': 0.0, 'quota_imponibile': 0.0},
]
risultato = storage.giornate_lavoro_agile_nel_mese(richieste, 'Mario', '2026-04')
print('Giornate agile nel mese (atteso 8):', risultato)
# Deve restituire 8 (solo la richiesta id=1: valida + lavoro_agile)
# id=2 è pasto (non conta), id=3 è respinta (non conta)
"
```

---

## Note importanti

- La funzione usa `r.get("giorni", 0) or 0` per gestire il caso in cui il campo  
  `giorni` fosse `None` (le richieste respinte possono avere campi mancanti).
- Le richieste con `stato == "respinta"` vengono escluse: una richiesta respinta  
  non produce effetti sul conteggio mensile, come stabilito dalla norma.
