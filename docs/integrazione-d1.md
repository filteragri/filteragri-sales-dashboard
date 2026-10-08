# Collegamento D1 reale alla dashboard v0 (fase B)

**Preservare integralmente il frontend approvato.** Nessuna rigenerazione del design; integrare solo la fonte dati.

## Stato al 9 ottobre 2026

La app v0 e' pubblicata sul progetto Vercel `filteragri-sales-dashboard`, con dati DEMO. Il GitHub `main` contiene soltanto README e docs, quindi il codice generato in v0 deve essere sincronizzato su un branch GitHub prima di effettuare modifiche esterne. Non creare un secondo progetto.

## Fonte e flusso

- `orders-collector-v2` raccoglie ordini dal sito tre volte al giorno (Cloudflare), scrive su D1 `sales_archive`.
- `filteragri-dashboard-sync` aggiorna `dashboard_daily` e `dashboard_product_monthly` 3 volte al giorno, con finestra 75 giorni.
- La dashboard Vercel legge il D1 tramite Cloudflare REST API **dal server Next.js**, non tramite endpoint pubblico del Collector.
- Il browser accede solo all'endpoint same-origin `/api/dashboard`; nessun token Cloudflare in codice client, log o repository.
- Non creare altri Worker, Cron, database o password personalizzate.
- Non disabilitare la protezione SSO/Deployment Protection del progetto Vercel; nessun dominio custom pubblico con dati commerciali reali.

## Variabili Vercel (SOLO server, da configurare dal proprietario)

```env
CLOUDFLARE_ACCOUNT_ID=<account-id>
CLOUDFLARE_D1_DATABASE_ID=<sales-archive-database-uuid>
CLOUDFLARE_API_TOKEN=<token-read-only>
```

Il token Cloudflare deve avere solo permesso `D1 Read`, associato all'account corretto. Non usare Global API Key, `D1 Write` o `NEXT_PUBLIC_*`. Non incollare il token nel prompt a v0.

## API server-side

Crea `app/api/dashboard/route.ts` e client `lib/d1-server.ts` con `fetch` POST a:

```text
https://api.cloudflare.com/client/v4/accounts/<account-id>/d1/database/<database-uuid>/query
```

`Authorization: Bearer <token>`, `Content-Type: application/json`, payload `{sql: "SELECT ...", params: []}`. Controllare HTTP `ok`, JSON `success` e `result[0].success`, restituendo `result[0].results`. Accettare soltanto query SELECT statiche e valide; mai SQL arbitrario passato dal browser. Cache dati 10-15 minuti lato server o equivalente, senza cache pubblica di dati cliente.

### Query giornaliera (aggregati esistenti)

```sql
SELECT giorno_italia AS day, ordini_validi AS orders,
       vendite_lorde AS gross,
       ROUND(vendite_lorde / 1.22, 2) AS net,
       aggiornato_il AS updatedAt
FROM dashboard_daily ORDER BY giorno_italia;
```

### Query prodotti e margini (gia' aggregati)

```sql
SELECT mese, product_code, prodotto, categoria, quantita,
       vendite_nette, costo_noto, vendite_con_costo, righe_senza_costo
FROM dashboard_product_monthly
ORDER BY mese, product_code;
```

Margine stimato su righe coperte da costo = (vendite_con_costo - costo_noto) / vendite_con_costo. Copertura = vendite_con_costo / vendite_nette. Il fatturato di queste righe e' SOLO prodotti: non va confrontato direttamente con le vendite ordine che comprendono trasporti.

### Query clienti (solo dati minimi necessari, nessuna PII diretta)

```sql
SELECT order_id AS id, giorno_italia AS day,
       customer_key AS customerKey, order_status_type AS status,
       order_total AS gross,
       ROUND(order_total / 1.22, 2) AS net
FROM dashboard_orders_valid
ORDER BY giorno_italia, order_id;
```

Circa 12.420 righe storiche; leggerle al massimo una volta per refresh, non a ogni click dei filtri. Se performance/costi sono elevati, in seguito creare un aggregato clienti invece di pesanti query. Il campo customerKey e' uno pseudonimo persistente, quindi l'app e l'API restano dietro protezione Vercel. Non restituire nome, indirizzo, PIVA, codice fiscale o email.

`customers.order_count` e' inaffidabile come totale storico perche' il Collector lo sovrascrive al singolo batch. Calcolare nuovi / returning / cohort dalle date degli ordini validi, in `lib/metrics.ts`.

### Query aggiornamento Collector

```sql
SELECT MAX(completed_at) AS lastOrderSync
FROM sync_runs
WHERE status = 'completed'
AND run_type IN ('scheduled_v2', 'manual_sync_v2');
```

Mostrare distintamente ultimo sync ordini e ultimo aggiornamento KPI (MAX aggiornato_il nella query giornaliera): il cron KPI puo' aggiornarsi anche se il Collector non importa nuovi ordini.

## Adattamento dell'app v0

- Preservare markup, styling, grafici, filtri e tutte le sezioni dell'app corrente.
- Sostituire `fetchDashboardData` con fetch same-origin a `/api/dashboard`, adattando gli alias SQL agli interface/type gia' presenti in `lib/dashboard-data.ts`.
- Il `DashboardProvider` oggi usa `DEMO_TODAY` fisso per i filtri: sostituire con giorno reale in Europe/Rome (usare Intl.DateTimeFormat con timeZone).
- `lib/metrics.ts` continua a calcolare sulla struttura coerente dei dati, senza modificare inutilmente le definizioni.
- Le sei card principali: netto stimato IVA esclusa (trasporti compresi), crescita YoY, ordini, scontrino medio netto, clienti attivi, % clienti di ritorno.
- Clienti: nuovi vs ritorno, retention 2025->2026, quota fatturato 2026 da clienti 2025, mediana giorni riacquisto.
- Prodotti: categorie e SKU, vendite prodotti, margine stimato, copertura costi.
- Tenere indicatori DEMO solo quando la modalita' e' DEMO, e rimuoverli solo dopo prima chiamata API reale riuscita; in caso di errore API mostrare errore, MAI fallback nascosto a valori falsi.
- Finestre anno corrente vs anno precedente equivalenti (YTD vs YTD, mese in corso vs stesso periodo).
- Interazioni sui dati ricevuti in SWR/cache, non nuovi accessi D1 a ogni filtro.

## Acceptance checks

1. Nessun dato demo presentato come live.
2. Il 7/10/2026 e l'8/10/2026 D1 aveva rispettivamente 18 e 9 ordini nella dashboard_orders_valid prima di eventuali cambi di stato successivi.
3. Le somme giornaliere e mensili coincidono con i dati D1, IVA 22% rimossa con /1.22; non e' un dato fiscale certificato.
4. V0 frontend visivamente invariato e con filtri funzionanti, su desktop/mobile.
5. D1 API token non compare nel codice client o GitHub; errori Cloudflare gestiti.
6. Cloudflare rows_read e Vercel tempo risposta verificati dopo il primo accesso reale.
7. Databox resta attivo finche' confronto finale KPI concluso.

## Prompt breve per v0

"Implementa la fase B leggendo docs/integrazione-d1.md. Preserva esattamente il frontend gia' approvato; modifica soltanto fonte dati, data corrente e segnali DEMO. Leggi D1 con API Next.js server-side e le 3 env Vercel CLOUDFLARE_ACCOUNT_ID, CLOUDFLARE_D1_DATABASE_ID, CLOUDFLARE_API_TOKEN, configurate separatamente da me. Nessun token nel prompt, nessun nuovo Worker/database/Cron, nessuna modifica al Collector. Non rendere i dati pubblici. Fai build, test dati reali e controlla che i filtri funzionino."
