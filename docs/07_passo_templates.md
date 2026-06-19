# Passo 6 – Modifica template e file statici

**Prerequisiti:** Passi 1–5 completati  
**Copre:** Aggiornamento interfaccia utente (normativa, riepilogo, form, JS)  
**Difficoltà:** ⭐ Bassa – solo HTML e una riga di JavaScript

---

## File da modificare

| File | Modifica |
|------|---------|
| `src/static/app.js` | Aggiungere `lavoro_agile` alla mappa dei campi |
| `src/templates/normativa.html` | Mostrare entrambe le tabelle (2025 e 2026) e nuove sezioni |
| `src/templates/riepilogo.html` | Mostrare il plafond per-riga invece che fisso |

> `src/templates/nuova_richiesta.html` e `src/templates/elenco.html` **non richiedono modifiche**:  
> il form usa il dizionario `categorie` (che ora include già `lavoro_agile` dopo il passo 1)  
> e l'elenco mostra i dati già presenti nella richiesta.

---

## Modifica A – `src/static/app.js`

### Codice attuale

```javascript
const CAMPI_PER_CATEGORIA = {
  trasferta_italia: "giorni",
  trasferta_estero: "giorni",
  pasto: "giorni",
  chilometrico: "km",
  alloggio: "notti",
};
```

### Modifica

Aggiungi la riga `lavoro_agile: "giorni"` alla mappa:

```javascript
const CAMPI_PER_CATEGORIA = {
  trasferta_italia: "giorni",
  trasferta_estero: "giorni",
  pasto: "giorni",
  lavoro_agile: "giorni",
  chilometrico: "km",
  alloggio: "notti",
};
```

**Perché:** Senza questa riga, quando l'utente seleziona "Indennità lavoro agile" nel form,  
il campo "Numero di giornate" rimane nascosto e il campo non viene inviato al server.

---

## Modifica B – `src/templates/normativa.html`

### Nuovo contenuto completo del file

Sostituisci l'**intero contenuto** di `src/templates/normativa.html` con:

```html
{% extends "base.html" %}
{% block titolo %}Normativa vigente{% endblock %}
{% block contenuto %}
<h1>Normativa vigente</h1>
<p class="nota">
  Il sistema applica le regole della <strong>{{ rules.RIFERIMENTO_NORMATIVO }}</strong>
  per le spese dal 01/01/2026, e della <strong>{{ rules.RIFERIMENTO_NORMATIVO_PREVIGENTE }}</strong>
  per le spese fino al 31/12/2025 (regime transitorio, Sezione 7).
</p>

<h2>Massimali di esenzione – dal 01/01/2026 ({{ rules.RIFERIMENTO_NORMATIVO }})</h2>
<table class="tabella">
  <thead>
    <tr>
      <th>Categoria</th>
      <th class="num">Massimale di esenzione</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_italia"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2026["trasferta_italia"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_estero"] }} (giorni 1–5)</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2026["trasferta_estero"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_estero"] }} (giorni 6–10, riduzione 10%)</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_ESTERO_RIDUZIONE_10) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_estero"] }} (giorni 11+, riduzione 20%)</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_ESTERO_RIDUZIONE_20) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["pasto"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2026["pasto"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["lavoro_agile"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2026["lavoro_agile"]) }} € / giorno
        (max {{ rules.MAX_GIORNI_LAVORO_AGILE_MENSILI }} gg/mese)</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["chilometrico"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_KM_2026) }} € / km</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["alloggio"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_NOTTE_2026) }} € / notte</td>
    </tr>
    <tr class="riga-evidenziata">
      <td><strong>Plafond mensile complessivo per dipendente</strong></td>
      <td class="num"><strong>{{ "%.2f"|format(rules.PLAFOND_MENSILE_2026) }} € / mese</strong></td>
    </tr>
  </tbody>
</table>

<h2>Massimali previgenti – fino al 31/12/2025 ({{ rules.RIFERIMENTO_NORMATIVO_PREVIGENTE }})</h2>
<table class="tabella">
  <thead>
    <tr>
      <th>Categoria</th>
      <th class="num">Massimale di esenzione</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_italia"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2025["trasferta_italia"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["trasferta_estero"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2025["trasferta_estero"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["pasto"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALI_GIORNALIERI_2025["pasto"]) }} € / giorno</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["chilometrico"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_KM_2025) }} € / km</td>
    </tr>
    <tr>
      <td>{{ rules.CATEGORIE["alloggio"] }}</td>
      <td class="num">{{ "%.2f"|format(rules.MASSIMALE_NOTTE_2025) }} € / notte</td>
    </tr>
    <tr class="riga-evidenziata">
      <td><strong>Plafond mensile complessivo per dipendente</strong></td>
      <td class="num"><strong>{{ "%.2f"|format(rules.PLAFOND_MENSILE_2025) }} € / mese</strong></td>
    </tr>
  </tbody>
</table>

<section class="scheda">
  <h2>Come avviene il calcolo</h2>
  <ol>
    <li>Si determina il massimale applicabile alla richiesta (massimale unitario × quantità),
        usando i valori vigenti alla data di sostenimento.</li>
    <li>La quota esente teorica è il minore tra l'importo richiesto e il massimale applicabile.</li>
    <li>La quota esente effettiva è limitata dalla capienza residua del plafond mensile del dipendente.</li>
    <li>La parte eccedente è quota imponibile e concorre alla formazione del reddito.</li>
  </ol>
</section>

<section class="scheda">
  <h2>Regole speciali dal 01/01/2026</h2>
  <ul>
    <li><strong>Trasferta estero &gt; 5 giorni:</strong> riduzione progressiva del massimale
        dalla 6ª giornata (−10%) e dall'11ª giornata (−20%).</li>
    <li><strong>Lavoro agile:</strong> massimo {{ rules.MAX_GIORNI_LAVORO_AGILE_MENSILI }}
        giornate esenti per mese di calendario per dipendente;
        le giornate eccedenti sono integralmente imponibili.</li>
    <li><strong>Incompatibilità lavoro agile / trasferta:</strong> nella stessa giornata
        un dipendente non può ottenere sia l'indennità di lavoro agile sia
        un'indennità di trasferta; la richiesta in conflitto è respinta integralmente.</li>
  </ul>
</section>
{% endblock %}
```

---

## Modifica C – `src/templates/riepilogo.html`

La pagina di riepilogo mostra la "barra di utilizzo del plafond" per ogni riga.  
Dopo il passo 5, ogni riga nel contesto ora include il campo `plafond` con il valore  
corretto per quel mese. Il template deve usare `riga.plafond` invece del generico `plafond`.

### Codice attuale (riga rilevante)

```html
<p class="nota">Plafond mensile di esenzione: {{ "%.2f"|format(plafond) }} € per dipendente.</p>
```

### Nuovo codice

Sostituisci solo la riga `<p class="nota">` con:

```html
<p class="nota">
  Plafond mensile di esenzione: <strong>{{ "%.2f"|format(plafond) }} €</strong> per dipendente
  (1.200,00 € per mesi fino al 31/12/2025; 1.400,00 € dal 01/01/2026).
</p>
```

Poi, nella riga che costruisce la barra di percentuale, aggiungi una cella con il plafond effettivo.  
Sostituisci il blocco `<td>` della colonna "Utilizzo plafond":

```html
      <td>
        <div class="barra-plafond">
          <div class="barra-plafond-riempimento {% if riga.percentuale_plafond >= 100 %}piena{% endif %}"
               style="width: {{ riga.percentuale_plafond }}%"></div>
        </div>
        <span class="percentuale">{{ riga.percentuale_plafond }}% su {{ "%.2f"|format(riga.plafond) }} €</span>
      </td>
```

---

## Come verificare le modifiche

### Verifica app.js

1. Avvia `flask --app src.app run`
2. Vai su `http://127.0.0.1:5000/nuova`
3. Seleziona "Indennità lavoro agile" dal menu categoria
4. Il campo "Numero di giornate" deve diventare visibile; km e notti devono sparire.

### Verifica normativa

1. Vai su `http://127.0.0.1:5000/normativa`
2. Devono apparire **due tabelle**: una per il 2026 e una per il 2025.
3. La tabella 2026 deve mostrare: pasto 10,00 €, lavoro agile 3,50 €, plafond 1.400,00 €.
4. La tabella 2025 deve mostrare: pasto 8,00 €, plafond 1.200,00 €.

### Verifica riepilogo

1. Registra alcune richieste con date 2025 e alcune con date 2026.
2. Vai su `http://127.0.0.1:5000/riepilogo`
3. Le righe del 2025 devono mostrare "su 1200,00 €" e le righe del 2026 "su 1400,00 €".
