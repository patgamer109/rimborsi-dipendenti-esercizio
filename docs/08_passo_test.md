# Passo 7 – Aggiornamento e aggiunta dei test

**Prerequisiti:** Passi 1–6 completati  
**Copre:** Verifica di conformità di tutte le NC e BUG identificati  
**Difficoltà:** ⭐⭐ Media

---

## Panoramica

Dopo i passi 1–6, alcuni test esistenti **falliranno** perché erano scritti per i valori 2025.  
Devono essere aggiornati. Inoltre vanno aggiunti test per tutte le nuove funzionalità.

---

## Parte A – Test da aggiornare

### A1 – `tests/test_calculator.py`

I test nella classe `TestMassimaleTeorico` usano date 2025 e si aspettano massimali 2025.  
Dopo il passo 2, `massimale_teorico()` è date-aware: con data 2025 restituisce i valori 2025,  
quindi i test **già passano senza modifica**.

L'unico test che fallisce è quello in `test_app.py` (vedi A2).

**Nessuna modifica richiesta** a `tests/test_calculator.py`.

### A2 – `tests/test_app.py` – `test_normativa_mostra_massimali_vigenti`

**Test attuale (riga finale del file):**

```python
def test_normativa_mostra_massimali_vigenti(client):
    testo = client.get("/normativa").get_data(as_text=True)
    assert "46.48" in testo
    assert "77.47" in testo
    assert "1200.00" in testo
```

**Perché fallisce:** La pagina normativa ora mostra i valori 2026 come primari  
(50,00 €, 85,00 €, 1.400,00 €) e quelli 2025 nella tabella "previgenti".  
Il test controlla che tutti e tre i valori siano presenti da qualche parte nella pagina.  
I valori 2025 appaiono ancora (tabella previgente), quindi **il test potrebbe ancora passare**.  
Tuttavia è buona pratica aggiornarlo per verificare anche i nuovi valori.

**Test aggiornato** (sostituisci il test esistente):

```python
def test_normativa_mostra_massimali_vigenti(client):
    testo = client.get("/normativa").get_data(as_text=True)
    # Valori 2026 (nuovi)
    assert "50.00" in testo
    assert "85.00" in testo
    assert "1400.00" in testo
    # Valori 2025 (previgenti, ancora visibili per il regime transitorio)
    assert "46.48" in testo
    assert "77.47" in testo
    assert "1200.00" in testo
```

---

## Parte B – Nuovi test per `tests/test_calculator.py`

Aggiungi le seguenti classi di test **in fondo** al file `tests/test_calculator.py`.

### B1 – Regime transitorio: massimali 2025 per date 2025

```python
class TestRegimeTransitorio:
    def test_pasto_2025_usa_massimale_8(self):
        """Caso 6.6: data 2025 → massimale pasto 8,00 (non 10,00)."""
        r = richiesta(data="2025-12-18", categoria="pasto", giorni=3, importo=24.0)
        assert calculator.massimale_teorico(r) == 24.0  # 3 × 8,00

    def test_pasto_2026_usa_massimale_10(self):
        r = richiesta(data="2026-01-05", categoria="pasto", giorni=3, importo=30.0)
        assert calculator.massimale_teorico(r) == 30.0  # 3 × 10,00

    def test_trasferta_italia_2025(self):
        r = richiesta(data="2025-10-01", categoria="trasferta_italia", giorni=2)
        assert calculator.massimale_teorico(r) == 92.96  # 2 × 46,48

    def test_trasferta_italia_2026(self):
        r = richiesta(data="2026-03-01", categoria="trasferta_italia", giorni=2)
        assert calculator.massimale_teorico(r) == 100.0  # 2 × 50,00

    def test_plafond_2025_e_1200(self):
        """Con data 2025 il plafond applicato è 1.200, non 1.400."""
        r = richiesta(data="2025-10-06", categoria="pasto", giorni=1, importo=8.0)
        esente, imponibile, _ = calculator.calcola(r, esente_gia_riconosciuta=1200.0)
        assert esente == 0.0
        assert imponibile == 8.0

    def test_plafond_2026_e_1400(self):
        """Con data 2026 il plafond applicato è 1.400, non 1.200."""
        r = richiesta(data="2026-01-10", categoria="pasto", giorni=1, importo=10.0)
        # esente_gia = 1200 → capienza = 1400-1200 = 200 → tutto esente
        esente, imponibile, _ = calculator.calcola(r, esente_gia_riconosciuta=1200.0)
        assert esente == 10.0
        assert imponibile == 0.0
```

### B2 – Riduzione progressiva trasferta estero

```python
class TestTrasfertaEsteroProgressiva:
    def test_5_giorni_nessuna_riduzione(self):
        """Caso 6.3: esattamente 5 giorni → nessuna riduzione."""
        r = richiesta(data="2026-03-01", categoria="trasferta_estero", giorni=5)
        assert calculator.massimale_teorico(r) == 425.0  # 5 × 85,00

    def test_6_giorni_riduzione_parziale(self):
        """Caso 6.2: 6 giorni → 5 pieni + 1 al 10%."""
        r = richiesta(data="2026-03-01", categoria="trasferta_estero", giorni=6)
        assert calculator.massimale_teorico(r) == 501.5  # 5×85 + 1×76,50

    def test_12_giorni(self):
        """Caso dalla norma (Sezione 4): 12 giorni."""
        r = richiesta(data="2026-03-09", categoria="trasferta_estero", giorni=12)
        # 5×85 + 5×76,50 + 2×68 = 425 + 382,50 + 136 = 943,50
        assert calculator.massimale_teorico(r) == 943.5

    def test_trasferta_estero_2025_no_riduzione(self):
        """Caso 6.7: data inizio 2025 → nessuna riduzione progressiva."""
        r = richiesta(data="2025-12-01", categoria="trasferta_estero", giorni=12)
        # Usa massimale 2025: 77,47 × 12 = 929,64
        assert calculator.massimale_teorico(r) == round(77.47 * 12, 2)
```

### B3 – Categoria lavoro agile

```python
class TestLavoroAgile:
    def test_massimale_senza_precedenti(self):
        """10 giorni richiesti, nessuno già rimborsato → ammesse 10, massimale 35,00."""
        r = richiesta(data="2026-04-01", categoria="lavoro_agile",
                      giorni=10, importo=35.0)
        esente, imponibile, d = calculator.calcola(
            r, esente_gia_riconosciuta=0.0, giorni_lavoro_agile_nel_mese=0
        )
        assert esente == 35.0
        assert imponibile == 0.0
        assert d["massimale_teorico"] == 35.0  # 10 × 3,50

    def test_caso_6_4_oltre_limite_mensile(self):
        """Caso 6.4: 15 giorni richiesti, già 0 → ammesse 12, massimale 42,00."""
        r = richiesta(data="2026-05-01", categoria="lavoro_agile",
                      giorni=15, importo=52.5)
        esente, imponibile, d = calculator.calcola(
            r, esente_gia_riconosciuta=0.0, giorni_lavoro_agile_nel_mese=0
        )
        assert d["massimale_teorico"] == 42.0  # 12 × 3,50
        assert esente == 42.0
        assert imponibile == 10.5

    def test_giornate_gia_parzialmente_usate(self):
        """8 giorni già rimborsati, 6 richiesti → ammesse 4 (12−8), massimale 14,00."""
        r = richiesta(data="2026-05-20", categoria="lavoro_agile",
                      giorni=6, importo=21.0)
        esente, imponibile, d = calculator.calcola(
            r, esente_gia_riconosciuta=0.0, giorni_lavoro_agile_nel_mese=8
        )
        assert d["massimale_teorico"] == 14.0  # 4 × 3,50
        assert esente == 14.0
        assert imponibile == 7.0

    def test_limite_mensile_esaurito(self):
        """12 giornate già rimborsate → giornate ammesse = 0, tutto imponibile."""
        r = richiesta(data="2026-05-25", categoria="lavoro_agile",
                      giorni=3, importo=10.5)
        esente, imponibile, d = calculator.calcola(
            r, esente_gia_riconosciuta=0.0, giorni_lavoro_agile_nel_mese=12
        )
        assert d["massimale_teorico"] == 0.0
        assert esente == 0.0
        assert imponibile == 10.5
```

---

## Parte C – Nuovi test per `tests/test_validator.py`

Aggiungi i seguenti test **in fondo** al file `tests/test_validator.py`.

```python
# ---------------------------------------------------------------------------
# Test regime transitorio
# ---------------------------------------------------------------------------

def test_lavoro_agile_data_2025_respinto():
    """lavoro_agile con data 2025 → categoria non riconosciuta."""
    ok, motivazione = validator.valida(
        richiesta(data="2025-12-01", categoria="lavoro_agile", giorni=3)
    )
    assert not ok
    assert motivazione == "categoria non riconosciuta"


def test_lavoro_agile_data_2026_valido():
    ok, motivazione = validator.valida(
        richiesta(data="2026-01-15", categoria="lavoro_agile", giorni=3)
    )
    assert ok


# ---------------------------------------------------------------------------
# Test BUG-1: data futura
# ---------------------------------------------------------------------------

def test_data_futura_respinta():
    ok, motivazione = validator.valida(
        richiesta(data="2099-01-01", categoria="pasto", giorni=1)
    )
    assert not ok
    assert motivazione == "data futura non ammessa"


# ---------------------------------------------------------------------------
# Test incompatibilità lavoro agile / trasferta
# ---------------------------------------------------------------------------

def _trasferta_valida(dipendente="Maria Rossi", data="2026-03-02",
                      categoria="trasferta_italia", giorni=5):
    return {
        "id": 99, "dipendente": dipendente, "data": data,
        "categoria": categoria, "stato": "valida",
        "importo": 200.0, "giorni": giorni,
        "quota_esente": 200.0, "quota_imponibile": 0.0,
    }


def _agile_valido(dipendente="Maria Rossi", data="2026-03-20", giorni=3):
    return {
        "id": 98, "dipendente": dipendente, "data": data,
        "categoria": "lavoro_agile", "stato": "valida",
        "importo": 10.5, "giorni": giorni,
        "quota_esente": 10.5, "quota_imponibile": 0.0,
    }


def test_caso_6_5_lavoro_agile_vs_trasferta_overlap():
    """Caso 6.5: trasferta 02/03-06/03, lavoro agile 06/03-08/03 → incompatibile."""
    trasferta = _trasferta_valida(data="2026-03-02", giorni=5)
    nuova = richiesta(data="2026-03-06", categoria="lavoro_agile", giorni=3)
    ok, motivazione = validator.valida(nuova, [trasferta])
    assert not ok
    assert motivazione == "incompatibilità lavoro agile / trasferta"


def test_lavoro_agile_vs_trasferta_nessun_overlap():
    """Trasferta 02/03-06/03, lavoro agile 07/03-09/03 → valido."""
    trasferta = _trasferta_valida(data="2026-03-02", giorni=5)
    nuova = richiesta(data="2026-03-07", categoria="lavoro_agile", giorni=3)
    ok, _ = validator.valida(nuova, [trasferta])
    assert ok


def test_trasferta_vs_lavoro_agile_overlap():
    """Direzione inversa: lavoro agile esistente, nuova trasferta in overlap."""
    agile = _agile_valido(data="2026-03-06", giorni=3)
    nuova = richiesta(data="2026-03-02", categoria="trasferta_italia", giorni=5)
    ok, motivazione = validator.valida(nuova, [agile])
    assert not ok
    assert motivazione == "incompatibilità lavoro agile / trasferta"


def test_incompatibilita_solo_per_dipendente_corretto():
    """Trasferta di un altro dipendente non blocca il lavoro agile."""
    trasferta = _trasferta_valida(dipendente="Luca Bianchi", data="2026-03-02", giorni=5)
    nuova = richiesta(data="2026-03-06", categoria="lavoro_agile", giorni=3)
    # nuova è di "Maria Rossi" (default), trasferta di "Luca Bianchi"
    ok, _ = validator.valida(nuova, [trasferta])
    assert ok


def test_richiesta_respinta_non_produce_incompatibilita():
    """Una richiesta risposta non blocca le nuove richieste."""
    trasferta_respinta = {
        **_trasferta_valida(data="2026-03-02", giorni=5),
        "stato": "respinta",
    }
    nuova = richiesta(data="2026-03-06", categoria="lavoro_agile", giorni=3)
    ok, _ = validator.valida(nuova, [trasferta_respinta])
    assert ok


def test_incompatibilita_non_applicata_a_date_2025():
    """Overlap tutto in 2025 → incompatibilità non si applica."""
    trasferta = {
        **_trasferta_valida(data="2025-12-01", giorni=5),
        "categoria": "trasferta_italia",
    }
    # Lavoro agile 2025 sarebbe respinto per regime transitorio,
    # ma proviamo con pasto per verificare che la logica non scatti
    nuova_pasto = richiesta(data="2025-12-03", categoria="pasto", giorni=1)
    ok, _ = validator.valida(nuova_pasto, [trasferta])
    assert ok
```

---

## Parte D – Nuovi test per `tests/test_app.py`

Aggiungi i seguenti test **in fondo** al file `tests/test_app.py`.

```python
# ---------------------------------------------------------------------------
# Helper per richieste 2026
# ---------------------------------------------------------------------------

def nuova_richiesta_2026(client, **campi):
    dati = {
        "dipendente": "Mario Verdi",
        "data": "2026-03-10",
        "categoria": "pasto",
        "importo": "20.00",
        "giorni": "2",
    }
    dati.update(campi)
    return client.post("/nuova", data=dati)


# ---------------------------------------------------------------------------
# Test regime transitorio end-to-end
# ---------------------------------------------------------------------------

def test_pasto_2025_usa_massimale_previgente(client):
    """Con data 2025 il massimale pasto è 8,00 € → 3 giorni × 8 = 24 €."""
    nuova_richiesta_pasto(client, importo="30.00", giorni="3")
    richieste = storage.carica()
    assert richieste[0]["quota_esente"] == 24.0
    assert richieste[0]["quota_imponibile"] == 6.0


def test_pasto_2026_usa_massimale_aggiornato(client):
    """Con data 2026 il massimale pasto è 10,00 € → 3 giorni × 10 = 30 €."""
    nuova_richiesta_2026(client, categoria="pasto", importo="30.00", giorni="3")
    richieste = storage.carica()
    assert richieste[0]["quota_esente"] == 30.0
    assert richieste[0]["quota_imponibile"] == 0.0


def test_plafond_2026_e_1400(client):
    """Il plafond del 2026 è 1.400 €, non 1.200 €."""
    nuova_richiesta_2026(
        client, categoria="alloggio", notti="8", importo="1300.00", giorni=""
    )
    nuova_richiesta_2026(client, categoria="pasto", importo="20.00", giorni="2")
    richieste = storage.carica()
    # capienza residua: 1400 - 1300 = 100
    assert richieste[1]["quota_esente"] == 20.0


# ---------------------------------------------------------------------------
# Test lavoro agile end-to-end
# ---------------------------------------------------------------------------

def test_lavoro_agile_2025_respinto(client):
    risposta = nuova_richiesta_2026(
        client, data="2025-12-01", categoria="lavoro_agile",
        importo="10.50", giorni="3"
    )
    testo = risposta.get_data(as_text=True)
    assert "respinta" in testo
    assert "categoria non riconosciuta" in testo


def test_lavoro_agile_2026_valido(client):
    risposta = nuova_richiesta_2026(
        client, categoria="lavoro_agile", importo="35.00", giorni="10"
    )
    assert "registrata" in risposta.get_data(as_text=True)
    richieste = storage.carica()
    assert richieste[0]["stato"] == "valida"
    assert richieste[0]["quota_esente"] == 35.0  # 10 × 3,50


def test_lavoro_agile_cap_mensile(client):
    """12 giornate usate: le successive 3 sono imponibili."""
    nuova_richiesta_2026(
        client, categoria="lavoro_agile", importo="42.00", giorni="12"
    )
    nuova_richiesta_2026(
        client, categoria="lavoro_agile", importo="10.50", giorni="3"
    )
    richieste = storage.carica()
    # Seconda richiesta: giornate ammesse = 0, tutto imponibile
    assert richieste[1]["quota_esente"] == 0.0
    assert richieste[1]["quota_imponibile"] == 10.5


# ---------------------------------------------------------------------------
# Test incompatibilità end-to-end
# ---------------------------------------------------------------------------

def test_incompatibilita_lavoro_agile_trasferta(client):
    """Trasferta 10/03-14/03, poi lavoro agile 14/03 → respinto."""
    nuova_richiesta_2026(
        client, categoria="trasferta_italia",
        importo="250.00", giorni="5", data="2026-03-10"
    )
    risposta = nuova_richiesta_2026(
        client, categoria="lavoro_agile",
        importo="3.50", giorni="1", data="2026-03-14"
    )
    testo = risposta.get_data(as_text=True)
    assert "respinta" in testo
    assert "incompatibilità lavoro agile / trasferta" in testo
```

---

## Parte E – Eseguire tutti i test

```bash
cd c:\prove\rimborsi-dipendenti-esercizio
pytest tests/ -v
```

**Risultato atteso dopo tutti i passi 1–7:**
- Tutti i test precedenti passano.
- Tutti i nuovi test passano.
- Nessun warning o errore di importazione.

---

## Checklist finale di conformità

| Non-conformità | Test di copertura |
|---------------|-----------------|
| NC-1 Massimali 2026 | `TestRegimeTransitorio.test_pasto_2026_*`, `test_pasto_2026_usa_massimale_aggiornato` |
| NC-2 Plafond 1.400 € | `TestRegimeTransitorio.test_plafond_2026_*`, `test_plafond_2026_e_1400` |
| NC-3 Lavoro agile | `TestLavoroAgile.*`, `test_lavoro_agile_*` |
| NC-4 Riduzione progressiva | `TestTrasfertaEsteroProgressiva.*` |
| NC-5 Incompatibilità | `test_caso_6_5_*`, `test_trasferta_vs_*`, `test_incompatibilita_*` |
| NC-6 Regime transitorio | `TestRegimeTransitorio.*`, `test_pasto_2025_usa_*` |
| BUG-1 Data futura | `test_data_futura_respinta` |
