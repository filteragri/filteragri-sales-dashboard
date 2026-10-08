# Filteragri Sales Dashboard — Architettura approvabile e brief v0
Aggiornato: 9 ottobre 2026 — fase MVP, uso interno, durata prevista circa 3 mesi.

## Architettura semplice

```
API Filteragri
   -> Cloudflare Worker orders-collector-v2 (00, 08, 16 UTC)
   -> D1 sales_archive: orders, order_lines, products_cache
   -> dashboard_orders_valid (view, ordini validi)
   -> Worker filteragri-dashboard-sync (01, 09, 17 UTC)
   -> dashboard_daily + dashboard_product_monthly
   -> dashboard_kpi_daily / dashboard_kpi_monthly (view)
   -> Cloudflare D1 REST API (token limitato a D1 Read, accesso solo server-side)
   -> Vercel / Next.js server API
   -> Frontend v0 (browser)
```

Nessun nuovo Worker, nessun nuovo Cron, niente collegamento diretto browser-D1. La sorgente dei dati è D1, non GitHub e non v0. Cambiare codice GitHub aggiorna la UI dopo deploy Vercel; NON fa partire una sincronizzazione dati.

## Tempistiche / affidabilita

- Collector ogni 8 ore; aggregazione un'ora dopo. Dashboard aggiornata 3 volte al giorno, NON real time; un ordine appena creato può impiegare fino a circa 9 ore per comparire.
- Il frontend legge i risultati già calcolati, senza rilanciare job di import.
- In UI mostrare *Ultimo aggiornamento KPI* (MAX dashboard_daily.aggiornato_il) e separatamente *Ultima sincronizzazione ordini completata* (MAX started_at/ completed_at da sync_runs con status completed). Segnalare se la raccolta o l'aggregazione è vecchia: aggiornare la tabella senza nuovi dati non equivale ad aggiornare gli ordini.
- dashboard_daily ricostruisce ultimi 75 giorni ogni run; dashboard_product_monthly ricostruisce i mesi toccati da quella finestra. Nessun backfill storico automatico. Modifiche ad ordini creati prima dei 75 giorni non si rifletteranno nei riepiloghi precedenti: limite accettato temporaneamente.
- Verificare una volta le metriche D1 rows_read / rows_written per il Worker di aggregazione, dopo la prima esecuzione; non ottimizzare alla cieca.

## Criteri numerici

- KPI ricavati dagli ordini e-commerce API; il bilancio include inoltre vendite manuali/WhatsApp e differenze di competenza/incasso di bonifici e contrassegni. Non pretendere corrispondenza perfetta.
- Validi: PAID, CLOSED, SHIPPED, PAYMENT_PENDING, altri stati non CANCELED. CANCELED sempre esclusi.
- Deduplicazione attuale view dashboard_orders_valid: stesso cliente_key + giorno Italia + stesso importo; priorità agli ordini non PAYMENT_PENDING. ATTENZIONE: la VIEW corrente **non confronta il contenuto delle righe prodotto**. Non affermare il contrario. Una verifica campionaria e un affinamento saranno possibili prima della messa in produzione.
- Vendite nette stimate (trasporto incluso): dashboard_kpi_daily/monthly = vendite lorde / 1.22. È una stima gestionale in assenza del dettaglio dell'IVA totale; non un dato fiscale certificato.
- Vendite prodotti, costi e margini commerciali stimati: dashboard_product_monthly; costi Channable attuali, copertura costi nota; eventuali coupon aggiuntivi non sempre ripartiti. **Non presentare le vendite prodotti come vendite complessive comprensive di spedizione**.
- Qualità dati: dashboard_product_monthly aggiornata da cron, non costi storici.

## KPI e interfaccia — MVP

**Overview** (massimo 6 KPI visibili immediatamente):
1. Vendite nette stimate IVA esclusa, trasporto incluso
2. Crescita YoY % rispetto allo stesso periodo dell'anno scorso
3. Ordini validi
4. Scontrino medio netto
5. Clienti unici attivi nel periodo
6. Quota clienti di ritorno nel periodo (+ numero)

Sotto: grafico vendite giornaliere/mensili; confronto degli anni 2024/2025/2026 con checkbox multi-selezione, mesi gennaio-dicembre, tooltip; margine stimato (+ copertura) come KPI distinto.

**Clienti**: nuovi vs di ritorno; quota fatturato 2026 originato da clienti attivi nel 2025; retention coorte 2025→2026 (al momento YTD parziale); mediana giorni fra acquisti distinti. Utilizzare customer_key dalla view ordini, MAI customers.order_count, che riflette solo l'ultimo batch.
- Nuovi nel periodo: primo acquisto valido osservato nello storico che parte dal 2024.
- Ritorno nel periodo: acquistato nel periodo e già prima della sua data d'inizio.
- Retention 2025→2026: n clienti attivi 2025 con almeno un ordine nel 2026 / n clienti attivi 2025; 2026 parziale.
- Fatturato clienti 2025 nel 2026: somma ordini 2026 dei customer_key presenti nel 2025 / vendite 2026.
- Tempo riacquisto: mediana tra due date di acquisto valide consecutive, evitando duplicati e specificando criterio.
Queste metriche **non sono ancora preaggregate**. Prima costruire UI mock; poi un endpoint server-side per le query. Misurare righe lette e aggiungere aggregazione clienti solo se necessario.

**Prodotti**: top SKU, categorie, quantità e margini stimati, dal riepilogo mensile; nessuna nuova JOIN del feed a ogni visita.

**Controlli**:
- Periodo card: oggi, ieri, 7 giorni, 30 giorni, questo mese, anno in corso, date custom (fino a oggi).
- Selettore annuale indipendente per grafico mensile: 2024, 2025, 2026, ogni combinazione (almeno una selezione).
- A mobile 390px, responsive sobrio, niente sidebar invadente; filtri chiari e leggibili, loading/error/empty states.
- Le card mutano per periodo; grafico anni mostra dati cumulati mese per mese e lascia vuoti i mesi futuri.
- L'anno corrente YTD va confrontato con lo stesso periodo dell'anno precedente, non intero mese/anno precedente.

## Sicurezza e costi

- In Vercel tenere CLOUDFLARE_API_TOKEN (permission D1 Read, DB/account specifici se possibile), CLOUDFLARE_ACCOUNT_ID e CLOUDFLARE_D1_DATABASE_ID come variabili SERVER-SIDE, mai NEXT_PUBLIC_*, mai nel browser o GitHub.
- Connessione REST: POST https://api.cloudflare.com/client/v4/accounts/{account_id}/d1/database/{database_id}/query.
- Solo SQL SELECT di riepilogo con parametri, validazione rigorosa periodi; nessun SQL arbitrario dalle richieste utente.
- Proteggere la dashboard vera; sviluppo con preview Vercel Authentication (nessuna password custom necessaria). Attenzione: su Hobby la protezione standard **non** protegge automaticamente un dominio produzione, quindi non rendere pubblici dati veri prima di una verifica delle impostazioni.
- Finché non collegato backend, solo dati demo chiaramente etichettati, nessun dato cliente reale e nessun valore demo presentato come KPI reale.
- In Next.js utilizzare server route con piccolo TTL cache (es. 10 minuti) se i test lo richiedono; dati aggiornati comunque solo 3 volte/die. Non chiedere D1 una volta per ciascuna card o ciascun clic.
- Mantenere Databox attivo finché i principali KPI della dashboard non sono stati confrontati.

## Prompt operativo per v0

Importa il repository privato GitHub `filteragri/filteragri-sales-dashboard` in v0. Leggi `docs/dashboard-specifiche.md` e questo file, poi crea una app Next.js (App Router) con TypeScript, Tailwind e componenti shadcn/ui. Progetta una dashboard gestionale italiana premium ma sobria, con navigazione Overview / Clienti / Prodotti, 6 KPI Overview, filtri temporali effettivamente funzionanti, line chart con confronto 2024/2025/2026 multi-selezione, tooltip, grafico giornaliero, sezioni clienti/cohort e prodotti/margine. Stile SaaS professionale, chiaro, poco decorativo, mobile curato.

**Per questa prima iterazione costruisci SOLO il frontend funzionante con dati mock esplicitamente DEMO**: niente chiamate Cloudflare, nessun token, nessun dato personale, nessuna API vera. Isola i dati con un modulo tipizzato `lib/dashboard-data.ts` e componenti che in una seconda fase potranno essere alimentati da API Next.js server-side. Tutti i controlli devono aggiornare visivamente metriche e grafici in modo coerente. Non inventare valori effettivi Filteragri; valori fittizi chiaramente identificati nel mock. Non introdurre login personalizzato, database, Cron aggiuntivi, Stripe, CMS o microservizi. Nel footer indica 'Demo UI — dati simulati'. Esegui build/lint e correggi eventuali errori. Non sovrascrivere il documento delle specifiche.
