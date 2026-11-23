# olivia-creator-gtm

Pipeline che trova creator Instagram italiani da attivare come partner di Olivia (app di nutrizione con coach AI) tramite codice creator. Obiettivo: ~30 creator attivi al mese.

La spec completa è in `docs/spec.md`, il piano in `docs/milestones.md`. **Leggile prima di ogni milestone.** Se il codice e la spec divergono, fermati e chiedi: non decidere da solo.

## Perimetro (non cambiarlo senza ok esplicito)
- Solo **Instagram**, solo **Italia** (creator e audience italiani).
- Target: nano/micro creator, **5k–100k follower**.
- Compenso: **codice creator**. Niente fee fisse nella logica.
- **Chi ha un'agenzia o un management non compare mai nei risultati.** È un filtro netto, non una penalità.
- Gli `unknown` (nessun segnale di agenzia) restano nei risultati, marcati "da verificare".

## Regole non negoziabili
1. **Mai inviare messaggi.** Niente DM, email o follow automatici. La pipeline produce bozze nello Sheet; l'invio è sempre manuale.
2. **Mai scraping autenticato.** Niente login Instagram, cookie o sessioni: solo dati pubblici via actor Apify senza login.
3. **Ogni chiamata Apify ha un limite di risultati e un limite di spesa** (`max_items` / `max_total_charge_usd` o equivalenti dell'actor), letti da config. Nessuna chiamata senza limite, nemmeno nei test manuali.
4. **I test non chiamano mai servizi esterni** (Apify, LLM, Google). Usano fixture in `tests/fixtures/`, ricavate dai dump reali in `data/raw/`.
5. **Lo score lo calcola il codice, mai l'LLM.** L'LLM restituisce campi strutturati; pesi e soglie stanno in `config/scoring.yaml`.
6. **Ogni citazione dell'LLM deve esistere.** Ogni campo classificato porta `post_id` + estratto; un estratto che non è substring (case/accenti-insensitive) dell'input viene scartato e il campo trattato come non provato.
7. **Esclusione per agenzia solo con segnale forte** (vedi sotto). I segnali deboli non escludono mai.
8. **Pesi, soglie, regex, hashtag, prompt e modelli stanno in `config/`**, mai hardcoded. Ogni riga in `scores` salva la versione di config e prompt usata.
9. **Niente segreti nel repo.** Tutto da `.env` (vedi `.env.example`). Mai stampare chiavi nei log.
10. **Nessuna dipendenza dal backend di Olivia.** Questo repo non importa codice né accede al DB di produzione; le conversioni entrano come import di file/API in sola lettura (milestone M7).

## Stack
Python 3.12 · `uv` · `typer` (CLI) · SQLite (`sqlite3` stdlib, niente ORM) · `apify-client` · `httpx` + `selectolax` (crawler link in bio) · `anthropic` (classificazione e bozze) · `pydantic` (validazione output LLM e config) · `gspread` (Sheet) · `pyyaml` · `langdetect` · `pytest` · `ruff`.

Nuove dipendenze solo se necessarie: motivale nel messaggio di commit.

## Struttura
```
gtm/
  cli.py              # typer: gtm <stage> [--limit N] [--dry-run]
  config.py           # carica e valida config/*.yaml con pydantic
  db.py               # schema, migrazioni semplici, helper per status
  apify.py            # wrapper unico: limiti obbligatori, salva il dataset grezzo in data/raw/ prima del parsing
  sources/            # hashtag.py, competitor.py, related.py
  prefilter.py
  agency/             # bio_rules.py, roster.py, domains.py, linkinbio.py, check.py
  enrich/             # profile.py, posts.py, comments.py, bio_crawler.py
  metrics.py
  classify/           # llm.py (+ prompt in config/prompts/)
  score.py
  sheet/sync.py
  outreach/draft.py
  report.py
  purge.py
config/   sources.yaml  filters.yaml  agency.yaml  scoring.yaml  llm.yaml  prompts/
tests/    fixtures/  golden/  test_*.py
data/     leads.db  raw/          # gitignored
docs/     spec.md  milestones.md
```

## Modello degli stage
Ogni stage è **idempotente**: seleziona i creator nello status di ingresso, li processa, li fa avanzare. Rilanciarlo non riscarica né ripaga nulla.

```
discovered → prefiltered | excluded
prefiltered → enriched → agency_checked | excluded
agency_checked → measured → classified → scored → shortlisted | parked
shortlisted → synced
```
- `excluded` ha sempre `exclude_reason` + `exclude_evidence`.
- `--limit N` conta i **creator nuovi** processati, non le righe scaricate.
- `--dry-run` non scrive su DB/Sheet e non chiama servizi a pagamento: stampa cosa farebbe e la stima di costo.
- Ogni run scrive una riga in `runs` (stage, volumi in/out, costo Apify, costo LLM, errori).

## Filtro agenzia
`agency_status ∈ {represented, independent, unknown}` + `agency_evidence` (fonte + testo).

**Segnali forti → `represented` → `excluded`:**
- handle nella tabella `agency_roster`;
- email business su un dominio classificato come agenzia in `domains`;
- bio con parola di management (`management`, `mgmt`, `managed by`, `rep by`, `booking`, `talent`, `agenzia`…, da `config/agency.yaml`) **seguita da** un'email o una @menzione che non è il creator;
- account menzionato in bio con categoria/bio da agenzia;
- link in bio verso pagina di management o form su dominio di agenzia;
- risposta del creator registrata nello Sheet.

**`independent`:** email su provider personale (gmail, icloud, outlook, hotmail, libero, yahoo…) o su un dominio che corrisponde al nome/sito del creator.

**Mai motivo di esclusione:** numero o grandezza delle sponsorizzazioni, "per collaborazioni scrivi a…" con email personale.

La classificazione di un dominio (crawl homepage + LLM) si fa una volta sola e resta in cache in `domains`.

## Dati e privacy
- I creator `excluded` restano solo con handle, motivo, data; si cancellano dopo 6 mesi (`purge`).
- Post e commenti di chi non è in shortlist si cancellano dopo 30 giorni.
- Non salvare dati di persone che non sono creator candidati (es. autori dei commenti): solo aggregati (quota emoji, lingua, ripetizioni).

## Sheet
Colonne dello script e colonne umane sono separate: lo script **non sovrascrive mai** le colonne umane (stato, date, note, codice). Upsert per `creator_id`. Le risposte "ha un'agenzia" lette dallo Sheet aggiornano il DB.

## Comandi
```
uv sync
uv run gtm --help
uv run gtm <stage> --limit 50 --dry-run
uv run pytest
uv run ruff check . && uv run ruff format --check .
```

## Test
- Ogni regex (bio agenzia, sponsorizzazioni, provider email) ha almeno 3 casi positivi e 3 negativi.
- `score.py`: test su pesi, penalità, esclusioni, ordinamento (a parità di score `independent` prima di `unknown`).
- Idempotenza: rilanciare uno stage due volte non duplica righe né chiamate (mock del wrapper Apify/LLM con contatore).
- Validazione citazioni LLM: estratto non presente → campo scartato.
- `tests/golden/`: profili etichettati a mano (fit sì/no, agenzia sì/no) per misurare prefiltri, filtro agenzia e score.

## Come lavorare
- **Una milestone per branch** (`feat/mN`), come da `docs/milestones.md`. Non anticipare codice delle milestone successive.
- **Prima il piano, poi il codice**: proponi file toccati, schema/migrazioni e test, e aspetta l'ok.
- Una milestone è chiusa quando: test verdi, `ruff` pulito, `--dry-run` funziona sullo stage nuovo, il risultato verificabile della milestone è dimostrato (output o report), `docs/` aggiornati se è cambiato qualcosa.
- Se un actor Apify non restituisce un campo che la spec dà per scontato, **non inventare un ripiego**: documentalo nel report della milestone e chiedi.
- Commit piccoli, messaggi in inglese, imperativo.
