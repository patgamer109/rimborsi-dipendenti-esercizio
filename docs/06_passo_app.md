# Passo 5 – Modifica `src/app.py`

**Prerequisiti:** Passi 1–4 completati  
**Copre:** NC-2 (plafond date-aware nel riepilogo), NC-3 (cablaggio lavoro agile), NC-5 (cablaggio incompatibilità)  
**Difficoltà:** ⭐⭐ Media – solo cablaggio di funzionalità già implementate nei passi precedenti

---

## Perché questo file va modificato

`app.py` è il "collante" che coordina validator, calculator e storage.  
Dopo i passi 1–4, tutte le nuove logiche esistono già, ma `app.py` non le usa ancora.  
Servono tre interventi:

1. **`_registra()`**: passare `richieste` a `validator.valida()` per abilitare il controllo  
   di incompatibilità (passo 4).
2. **`_registra()`**: calcolare le giornate di lavoro agile già rimborsate nel mese e  
   passarle a `calculator.calcola()` (passi 2–3).
3. **`riepilogo()`**: utilizzare il plafond corretto per ogni mese (1.200 € per 2025,  
   1.400 € per 2026) invece del valore fisso `rules.PLAFOND_MENSILE`.

---

## Modifica 1 – `_registra()`: cablaggio validator e calculator

### Codice attuale

```python
def _registra(form):
    """Valida, calcola e registra una nuova richiesta. Restituisce la richiesta salvata."""
    richieste = storage.carica()
    richiesta = {
        "id": storage.prossimo_id(richieste),
        "dipendente": (form.get("dipendente") or "").strip(),
        "data": form.get("data") or "",
        "categoria": form.get("categoria") or "",
        "importo": _numero(form.get("importo")),
        "giorni": _intero(form.get("giorni")),
        "km": _numero(form.get("km")),
        "notti": _intero(form.get("notti")),
    }
    ok, motivazione = validator.valida(richiesta)
    if ok:
        gia_riconosciuta = storage.esente_riconosciuta_nel_mese(
            richieste, richiesta["dipendente"], storage.mese(richiesta)
        )
        esente, imponibile, dettaglio = calculator.calcola(richiesta, gia_riconosciuta)
        richiesta.update(
            stato="valida",
            motivazione="",
            quota_esente=esente,
            quota_imponibile=imponibile,
            dettaglio=dettaglio,
        )
    else:
        richiesta.update(
            stato="respinta",
            motivazione=motivazione,
            quota_esente=0.0,
            quota_imponibile=0.0,
            dettaglio=None,
        )
    richieste.append(richiesta)
    storage.salva(richieste)
    return richiesta
```

### Nuovo codice

Sostituisci l'intera funzione `_registra` con:

```python
def _registra(form):
    """Valida, calcola e registra una nuova richiesta. Restituisce la richiesta salvata."""
    richieste = storage.carica()
    richiesta = {
        "id": storage.prossimo_id(richieste),
        "dipendente": (form.get("dipendente") or "").strip(),
        "data": form.get("data") or "",
        "categoria": form.get("categoria") or "",
        "importo": _numero(form.get("importo")),
        "giorni": _intero(form.get("giorni")),
        "km": _numero(form.get("km")),
        "notti": _intero(form.get("notti")),
    }
    # Passa le richieste esistenti al validator per il controllo di incompatibilità
    ok, motivazione = validator.valida(richiesta, richieste)
    if ok:
        gia_riconosciuta = storage.esente_riconosciuta_nel_mese(
            richieste, richiesta["dipendente"], storage.mese(richiesta)
        )
        # Per lavoro_agile: recupera le giornate già rimborsate nel mese
        giorni_agile = 0
        if richiesta["categoria"] == "lavoro_agile":
            giorni_agile = storage.giornate_lavoro_agile_nel_mese(
                richieste, richiesta["dipendente"], storage.mese(richiesta)
            )
        esente, imponibile, dettaglio = calculator.calcola(
            richiesta, gia_riconosciuta, giorni_agile
        )
        richiesta.update(
            stato="valida",
            motivazione="",
            quota_esente=esente,
            quota_imponibile=imponibile,
            dettaglio=dettaglio,
        )
    else:
        richiesta.update(
            stato="respinta",
            motivazione=motivazione,
            quota_esente=0.0,
            quota_imponibile=0.0,
            dettaglio=None,
        )
    richieste.append(richiesta)
    storage.salva(richieste)
    return richiesta
```

**Cosa è cambiato:**
- `validator.valida(richiesta)` → `validator.valida(richiesta, richieste)`
- Nuovo blocco `if richiesta["categoria"] == "lavoro_agile":` per calcolare `giorni_agile`
- `calculator.calcola(richiesta, gia_riconosciuta)` → `calculator.calcola(richiesta, gia_riconosciuta, giorni_agile)`

---

## Modifica 2 – Aggiungere helper `_plafond_per_mese()`

Aggiungi questa funzione in cima al file, dopo le funzioni `_numero` e `_intero` esistenti  
e prima di `_registra`:

```python
def _plafond_per_mese(mese_str):
    """Plafond mensile applicabile al mese dato (formato AAAA-MM).

    Regime transitorio: 1.200 € per mesi fino a 2025-12, 1.400 € da 2026-01.
    """
    return rules.PLAFOND_MENSILE_2026 if mese_str >= "2026-01" else rules.PLAFOND_MENSILE_2025
```

---

## Modifica 3 – `riepilogo()`: plafond date-aware

### Codice attuale (parte rilevante)

```python
    righe = [
        {
            "mese": mese,
            "dipendente": dipendente,
            "esente": dati["esente"],
            "imponibile": dati["imponibile"],
            "richieste": dati["richieste"],
            "percentuale_plafond": min(
                round(dati["esente"] / rules.PLAFOND_MENSILE * 100), 100
            ),
        }
        for (mese, dipendente), dati in sorted(gruppi.items(), reverse=True)
    ]
    return render_template(
        "riepilogo.html", righe=righe, plafond=rules.PLAFOND_MENSILE
    )
```

### Nuovo codice (parte rilevante)

Sostituisci il blocco `righe = [...]` e il `return render_template(...)` con:

```python
    righe = [
        {
            "mese": mese,
            "dipendente": dipendente,
            "esente": dati["esente"],
            "imponibile": dati["imponibile"],
            "richieste": dati["richieste"],
            "plafond": _plafond_per_mese(mese),
            "percentuale_plafond": min(
                round(dati["esente"] / _plafond_per_mese(mese) * 100), 100
            ),
        }
        for (mese, dipendente), dati in sorted(gruppi.items(), reverse=True)
    ]
    return render_template(
        "riepilogo.html", righe=righe, plafond=rules.PLAFOND_MENSILE_2026
    )
```

**Cosa è cambiato:**
- Aggiunto il campo `"plafond"` per riga (sarà usato dal template al passo 6).
- `percentuale_plafond` ora usa `_plafond_per_mese(mese)` invece del valore fisso.
- Il `plafond` passato al template per il messaggio di intestazione resta `PLAFOND_MENSILE_2026` (1.400 €),  
  in quanto è il valore corrente. Il template mostrerà comunque il plafond per-riga.

---

## Riepilogo di tutte le modifiche in `app.py`

| Posizione nel file | Modifica |
|---|---|
| Dopo `_intero()`, prima di `_registra()` | Aggiungere la funzione `_plafond_per_mese()` |
| Dentro `_registra()`, riga `validator.valida(richiesta)` | Aggiungere secondo argomento `richieste` |
| Dentro `_registra()`, blocco `if ok:` | Aggiungere calcolo `giorni_agile` e passarlo a `calculator.calcola()` |
| Dentro `riepilogo()`, costruzione della lista `righe` | Aggiungere campo `plafond` e aggiornare `percentuale_plafond` |
| Dentro `riepilogo()`, `return render_template(...)` | Cambiare `rules.PLAFOND_MENSILE` in `rules.PLAFOND_MENSILE_2026` |

---

## Come verificare che la modifica sia corretta

### Avvia l'applicazione

```bash
cd c:\prove\rimborsi-dipendenti-esercizio
flask --app src.app run
```

### Verifica manuale

1. Apri `http://127.0.0.1:5000/nuova`
2. Seleziona la categoria "Indennità lavoro agile" – il campo "Numero di giornate" deve apparire.
3. Inserisci: dipendente "Mario Rossi", data "2025-12-01", categoria "Indennità lavoro agile",  
   importo 10.50, giorni 3 → deve essere **respinta** con "categoria non riconosciuta".
4. Inserisci: dipendente "Mario Rossi", data "2026-02-01", categoria "Indennità lavoro agile",  
   importo 35.00, giorni 10 → deve essere **valida**, esente = 35,00 (10 × 3,50).
5. Inserisci: dipendente "Mario Rossi", data "2026-02-01", categoria "Indennità lavoro agile",  
   importo 10.50, giorni 3 → deve essere **valida**, ma solo 2 giornate ammesse  
   (12 − 10 = 2), esente = 7,00, imponibile = 3,50.

### Verifica via test

```bash
pytest tests/ -v
```

Tutti i test esistenti devono continuare a passare.  
I nuovi test saranno aggiunti al passo 7.
