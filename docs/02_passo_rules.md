# Passo 1 – Modifica `src/rules.py`

**Prerequisiti:** nessuno  
**Copre:** NC-1, NC-2, NC-3 (parziale), NC-6 (parziale)  
**Difficoltà:** ⭐ Bassa – solo aggiunta di costanti

---

## Perché questo file va modificato

`rules.py` è il "libro delle regole" del sistema: contiene tutti i valori numerici usati dal calcolatore.  
Attualmente contiene solo i valori della Circolare 41/2024 (anno 2025).  
Dobbiamo aggiungere:

1. I valori aggiornati per il 2026 (nuovi massimali e plafond).
2. Una data-soglia (`DATA_DECORRENZA_2026`) che permetta al calcolatore di capire quale set di valori applicare.
3. La costante per il limite mensile di 12 giorni del lavoro agile.
4. La nuova categoria `lavoro_agile` nel dizionario `CATEGORIE` e nella tupla `CATEGORIE_A_GIORNATE`.
5. I moltiplicatori per la riduzione progressiva delle trasferte estere.

---

## Contenuto attuale del file

```python
"""Parametri normativi vigenti per il calcolo dei rimborsi spese.

Fonte: Circolare MEF n. 41/2024, in vigore per l'anno 2025.
"""

MASSIMALI_GIORNALIERI = {
    "trasferta_italia": 46.48,
    "trasferta_estero": 77.47,
    "pasto": 8.00,
}

MASSIMALE_KM = 0.42
MASSIMALE_NOTTE = 150.00
PLAFOND_MENSILE = 1200.00

CATEGORIE = {
    "trasferta_italia": "Trasferta in Italia",
    "trasferta_estero": "Trasferta all'estero",
    "pasto": "Rimborso pasto",
    "chilometrico": "Rimborso chilometrico",
    "alloggio": "Rimborso alloggio",
}

CATEGORIE_A_GIORNATE = ("trasferta_italia", "trasferta_estero", "pasto")

RIFERIMENTO_NORMATIVO = "Circolare MEF n. 41/2024"
```

---

## Nuovo contenuto completo del file

Sostituisci l'**intero contenuto** di `src/rules.py` con il seguente:

```python
"""Parametri normativi per il calcolo dei rimborsi spese.

Fonti:
  - Circolare MEF n. 41/2024: regole per l'anno 2025
    (data di sostenimento <= 31/12/2025)
  - Circolare MEF n. 18/2026: regole per l'anno 2026
    (data di sostenimento >= 01/01/2026)
"""

from datetime import date

# ---------------------------------------------------------------------------
# Data-soglia per il regime transitorio (Sezione 7, Circ. 18/2026)
# ---------------------------------------------------------------------------
DATA_DECORRENZA_2026 = date(2026, 1, 1)

# ---------------------------------------------------------------------------
# Massimali – Circolare 41/2024 (applicabili fino al 31/12/2025)
# ---------------------------------------------------------------------------
MASSIMALI_GIORNALIERI_2025 = {
    "trasferta_italia": 46.48,
    "trasferta_estero": 77.47,
    "pasto": 8.00,
}
MASSIMALE_KM_2025 = 0.42
MASSIMALE_NOTTE_2025 = 150.00
PLAFOND_MENSILE_2025 = 1200.00

# ---------------------------------------------------------------------------
# Massimali – Circolare 18/2026 (applicabili dal 01/01/2026)
# ---------------------------------------------------------------------------
MASSIMALI_GIORNALIERI_2026 = {
    "trasferta_italia": 50.00,
    "trasferta_estero": 85.00,
    "pasto": 10.00,
    "lavoro_agile": 3.50,
}
MASSIMALE_KM_2026 = 0.45
MASSIMALE_NOTTE_2026 = 170.00
PLAFOND_MENSILE_2026 = 1400.00

# Limite mensile di giornate di lavoro agile esenti (Sezione 3, Circ. 18/2026)
MAX_GIORNI_LAVORO_AGILE_MENSILI = 12

# Riduzione progressiva trasferte estere >5 giorni (Sezione 4, Circ. 18/2026)
# giorni 1-5: pieno 85,00 €
# giorni 6-10: -10% → 76,50 €
# giorni 11+: -20% → 68,00 €
MASSIMALE_ESTERO_RIDUZIONE_10 = round(85.00 * 0.90, 2)   # 76.50
MASSIMALE_ESTERO_RIDUZIONE_20 = round(85.00 * 0.80, 2)   # 68.00

# ---------------------------------------------------------------------------
# Alias "correnti" puntano ai valori 2026 (usati da template e riepilogo)
# ---------------------------------------------------------------------------
MASSIMALI_GIORNALIERI = MASSIMALI_GIORNALIERI_2026
MASSIMALE_KM = MASSIMALE_KM_2026
MASSIMALE_NOTTE = MASSIMALE_NOTTE_2026
PLAFOND_MENSILE = PLAFOND_MENSILE_2026

# ---------------------------------------------------------------------------
# Categorie riconosciute
# ---------------------------------------------------------------------------
CATEGORIE = {
    "trasferta_italia": "Trasferta in Italia",
    "trasferta_estero": "Trasferta all'estero",
    "pasto": "Rimborso pasto",
    "chilometrico": "Rimborso chilometrico",
    "alloggio": "Rimborso alloggio",
    "lavoro_agile": "Indennità lavoro agile",
}

# Categorie che richiedono il campo "giorni" nella richiesta
CATEGORIE_A_GIORNATE = (
    "trasferta_italia",
    "trasferta_estero",
    "pasto",
    "lavoro_agile",
)

# Categorie trasferta (usate per il controllo incompatibilità con lavoro agile)
CATEGORIE_TRASFERTA = ("trasferta_italia", "trasferta_estero")

RIFERIMENTO_NORMATIVO = "Circolare MEF n. 18/2026"
RIFERIMENTO_NORMATIVO_PREVIGENTE = "Circolare MEF n. 41/2024"
```

---

## Come verificare che la modifica sia corretta

Apri un terminale nella cartella del progetto ed esegui:

```bash
python -c "
from src import rules
from datetime import date
print('OK: DATA_DECORRENZA_2026 =', rules.DATA_DECORRENZA_2026)
print('OK: MASSIMALI 2025 pasto =', rules.MASSIMALI_GIORNALIERI_2025['pasto'])   # atteso: 8.0
print('OK: MASSIMALI 2026 pasto =', rules.MASSIMALI_GIORNALIERI_2026['pasto'])   # atteso: 10.0
print('OK: MASSIMALI 2026 lavoro_agile =', rules.MASSIMALI_GIORNALIERI_2026['lavoro_agile'])  # atteso: 3.5
print('OK: PLAFOND 2025 =', rules.PLAFOND_MENSILE_2025)   # atteso: 1200.0
print('OK: PLAFOND 2026 =', rules.PLAFOND_MENSILE_2026)   # atteso: 1400.0
print('OK: MAX giorni lavoro agile =', rules.MAX_GIORNI_LAVORO_AGILE_MENSILI)  # atteso: 12
print('OK: riduzione 10% estero =', rules.MASSIMALE_ESTERO_RIDUZIONE_10)  # atteso: 76.5
print('OK: riduzione 20% estero =', rules.MASSIMALE_ESTERO_RIDUZIONE_20)  # atteso: 68.0
print('OK: lavoro_agile in CATEGORIE =', 'lavoro_agile' in rules.CATEGORIE)
print('OK: lavoro_agile in CATEGORIE_A_GIORNATE =', 'lavoro_agile' in rules.CATEGORIE_A_GIORNATE)
"
```

Tutti i valori devono corrispondere a quelli indicati nei commenti.

---

## Cosa NON cambiare

- Non rimuovere le costanti `MASSIMALI_GIORNALIERI_2025`, `PLAFOND_MENSILE_2025` ecc.:  
  servono al calcolatore per applicare il regime transitorio (passo 2).
- Non rinominare `CATEGORIE`, `CATEGORIE_A_GIORNATE`, `MASSIMALE_KM`, `MASSIMALE_NOTTE`, `PLAFOND_MENSILE`:  
  sono usati dai template HTML e da altre parti del codice.
