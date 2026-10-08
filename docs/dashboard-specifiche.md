# FILTERAGRI — Sales Dashboard | Specifiche MVP

**Scopo:** dashboard commerciale temporanea, 3 mesi, rapida e con costi operativi bassissimi. Frontend realizzato tramite **v0**, repository privato GitHub, deploy Vercel. Fonte Cloudflare D1 `sales_archive`. Databox rimane attivo fino alla validazione dei numeri.

## Definizione delle vendite
- Base: ordini e-commerce da API Filteragri, NON vendite complete a bilancio; quelle includono anche preventivi WhatsApp e differenze di registrazione di bonifici/contrassegni.
- Escludere sempre `CANCELED`. Includere `PAID`, `SHIPPED`, `CLOSED`, `PAYMENT_PENDING`.
- Escludere duplicati nello stesso giorno italiano, cliente e importo; nella view corrente il criterio è **cliente + giorno + totale** e preferisce un ordine non pending. **Attenzione**: per ora NON confronta la firma prodotti, NON asserire che lo faccia.
- Vendite nette stimate IVA esclusa e trasporti inclusi: al momento `order_total / 1.22`, affidabile come stima operativa per gli ordini standard al 22%; non chiamarlo dato fiscale.
- Confronti 2025/2026 su ordini sito, non sul bilancio; 2024 disponibile nello storico.
- Mese corrente vs anno precedente: confronto solo giorni corrispondenti, non mese pieno.

## KPI principali (mostrare in header massimo 6)
1. **Vendite nette stimate (€)** (prodotti + trasporto IVA esclusa).
2. **Crescita rispetto allo stesso periodo dell'anno precedente (%)**.
3. **Ordini validi** (annullati e duplicati esclusi).
4. **Scontrino medio netto (€)** = vendite nette stimate / ordini validi.
5. **Clienti attivi** = `COUNT(DISTINCT customer_key)` con almeno un ordine valido nel periodo.
6. **Clienti di ritorno (%)** = clienti attivi che avevano comprato almeno una volta *prima* dell'inizio del periodo / clienti attivi nel periodo. Accanto numero assoluto dei clienti di ritorno.

## KPI secondari
- **Margine commerciale stimato (%)**: da `dashboard_product_monthly`, con copertura costi; mostrato a parte, non fra i sei KPI principali. Costi correnti Channable, non necessariamente costi storici, coupon carrello non necessariamente ripartiti. Prodotto-only, non confondere con vendite ordine trasporto incluso.
- **Nuovi clienti** nel periodo = primo acquisto osservato nello storico registrato >= 2024-01-01.
- **Retention 2025 → 2026 (%)** = clienti 2025 che riacquistano nel 2026 / clienti 2025; 2026 YTD, quindi metrica parziale. Mostrare anche numero assoluto.
- **Quota fatturato 2026 da clienti 2025 (%)** = vendite nette stimate 2026 dei clienti che hanno almeno un ordine valido nel 2025 / tutte le vendite nette stimate 2026.
- **Tempo di riacquisto (giorni)**: mediana dei giorni tra ordini validi consecutivi dello stesso cliente; l'eventuale media come dato secondario. Escludere intervalli non significativi/duplicati e spiegare il criterio.
- **Top categorie e prodotti**: classifica per vendite prodotti nette (non includere spedizioni nel totale categorie), quantità, copertura dei costi, opzionalmente margine stimato.

## UI/UX richiesta
- Periodi selezionabili: oggi, ieri, 7 giorni, 30 giorni, mese corrente, YTD, intervallo personalizzato con date picker; badge *periodo parziale* se appropriato.
- Confronto anni 2024, 2025, 2026, accensione/spegnimento singole serie e tutte insieme. Grafico mensile con asse gennaio-dicembre e tooltip con valori.
- Grafico giornaliero quando selezionati intervalli brevi.
- Scheda **Clienti e riacquisto** separata dai KPI principali: nuovi vs ritorno, retention cohort 2025→2026, quota fatturato clienti 2025, tempo al riacquisto.
- Mobile responsive anche a 390px; layout SaaS sobrio, professionale, non AI-looking. Controlli realmente funzionanti, loading/empty/error.
- Dashboard privata in produzione. Nessuna credenziale D1 nel client/browser e nessun dato commerciale o personale reale pubblicato su URL aperti. Previews con dati di esempio chiaramente etichettati e senza PII.

## Dati già presenti in Cloudflare
- `dashboard_orders_valid`: view ordini validi, giorno Italia, customer_key, order_total e status.
- `dashboard_daily`: giorni, ordini validi, vendite lorde, scontrino medio lordo; aggiornata dal Cron.
- `dashboard_monthly`: view riepilogo lordo.
- `dashboard_kpi_daily` e `dashboard_kpi_monthly`: vendite IVA esclusa stimate dividendo per 1.22.
- `dashboard_product_monthly`: aggregato mensile per codice prodotto, quantità, vendite nette prodotto, costi noti, copertura.
- `orders`, `order_lines`, `products_cache`, `customers`: dati raw.
- `filteragri-dashboard-sync`: Worker Cloudflare cron `0 1,9,17 * * *`, refresh 75 giorni e mesi prodotto toccati.
- Collector `orders-collector-v2`: import manuale verificato, stabilità cron ancora da controllare.
- **Importante:** `customers.order_count` NON è il numero storico degli ordini: il Collector lo aggiorna soltanto con gli ordini dell'ultimo batch. I clienti si calcolano dagli ordini validi.

## Costi/performance
- Dati di aggregazione precomputati (non tutte le JOIN su richiesta per ogni visitatore). Niente query massive ogni caricamento. Le metriche cliente potranno richiedere un nuovo aggregato/refresh leggero sul D1.
- Sviluppo per fasi: (A) v0 UI interattiva con preview chiaramente etichettata, (B) API server-side e KPI clienti, (C) test contro D1/Databox, (D) rilascio privato.
- Nessun nuovo Worker o cron prima della fase B, se non indispensabile.

## Stato verificato
- D1 `dashboard_daily`: 1009 giorni, 12420 ordini, vendite lorde €1.745.280,45 nello storico 2024–08/10/2026 (dato prima dei refresh ulteriori).
- D1 `dashboard_product_monthly` nel 2026 gen–set: copertura costi delle vendite prodotto 95,9%-99,2%; margine stimato mese 41,1%-48,9%.
- Fatturato netto API gennaio-settembre:
  - 2025 €: 42272.56, 35461.28, 42085.56, 36675.09, 40980.44, 38958.92, 37350.95, 32911.80, 34882.75.
  - 2026 €: 37653.22, 39589.61, 69994.48, 61108.59, 54872.77, 44588.18, 68171.82, 51086.98, 67185.30.
- Il 2024 e i mesi successivi a settembre 2026 nel frontend v0 preview devono restare `null` finché non disponibili dall'API. Non mischiare dati di bilancio con dati API.
