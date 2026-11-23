# Pipeline GTM creator — Spec tecnica v0

Data: 2026-11-10 · Owner: Matteo · Stato: **bozza da validare**

## 0. Perimetro (deciso)
- **Obiettivo**: ~30 creator attivi al mese che promuovono Olivia sui propri canali.
- **Mercato**: solo Italia (creator italiani, audience italiana).
- **Piattaforma**: solo Instagram.
- **Compenso**: codice creator (nessuna fee fissa).
- **Agenzie**: filtro netto. Chi ha un'agenzia o un management non compare mai nei risultati. Gli `unknown` restano, marcati "da verificare".
- **Target conseguente**: nano e micro creator, circa 5k–100k follower.

## 1. Imbuto e volumi (ipotesi da ricalibrare dopo il primo mese)
| Stage | Volume/mese | Assunzione |
|---|---|---|
| Profili scoperti (discovery) | ~5.000 | hashtag + concorrenti + espansione da creator simili |
| Passano i pre-filtri (follower, lingua, attività, agenzia da bio/roster) | ~1.500 | ~30% |
| Enrich completo + metriche + classificazione | ~1.500 | |
| Shortlist (score ≥ soglia, non `represented`) | ~200 | ~13% |
| Contattati | ~150–200 | 5–8 contatti al giorno, a mano |
| Rispondono | ~40–50 | ~25% (contatto diretto, micro) |
| Attivi con codice | ~30 | ~60–70% di chi risponde |

Se il tasso di risposta reale è più basso, si alza la discovery, non la cadenza dei DM.

## 2. Flusso
```
discover → prefilter → enrich → agency_check → metrics → classify → score
        → shortlist (Sheet) → outreach (manuale) → deal (codice) → tracking → performance
        → feedback: i creator che convertono diventano seed per la discovery
```
Ogni stage è **idempotente**: lavora sulle righe nello status di ingresso, le fa avanzare, e rilanciarlo non riscarica né ripaga nulla.

## 3. Discovery (Apify)
Fonti, in ordine di valore:
1. **Creator che hanno già sponsorizzato concorrenti** (Yazio, MyFitnessPal, Lifesum, Fastic, ecc.): post con tag o menzione del brand + caption con "codice", "sconto", `#adv`. Actor: hashtag scraper e scraper dei post taggati del brand. Si registrano brand e data dell'ultima collaborazione (indizio di possibile esclusiva in corso).
2. **Hashtag italiani di nicchia**: `#ricettefit`, `#mealprep`, `#deficitcalorico`, `#ricetteproteiche`, `#colazionefit`, `#dietaflessibile`, `#nutrizione`… Lista in `sources.yaml`, con cursore per hashtag per scavare verso i post più vecchi invece di ricaricare sempre gli stessi recenti.
3. **Espansione da somiglianza**: dai creator in shortlist o attivi → profili correlati, account taggati nelle collaborazioni, commentatori ricorrenti che sono anch'essi creator. Il campo "related profiles" dei profile scraper non è sempre affidabile: va verificato con un run di prova, ed eventualmente si usa un actor dedicato.

Ogni scoperta salva `source` e `query` in `source_hits`, così si misura quali fonti portano creator che poi convertono.

Actor di partenza (da verificare con run di prova su 50 profili prima di scegliere): `apify/instagram-hashtag-scraper`, `apify/instagram-profile-scraper`, `apify/instagram-post-scraper` / `apify/instagram-scraper`, `apify/instagram-comment-scraper`. I dataset Apify si scaricano subito e si salvano nel DB.

## 4. Pre-filtri (gratis o quasi, prima dell'enrich)
- follower 5k–100k (con un margine 4k–120k per la crescita);
- lingua italiana (bio + caption, langdetect);
- ultimo post ≤ 30 giorni;
- **agenzia, primo passaggio**: handle presente nella blacklist dei roster delle agenzie, oppure regex sulla bio con segnale forte (§5).

## 5. Filtro agenzia
`agency_status ∈ {represented, independent, unknown}` + `agency_evidence` (fonte e testo).

**Esclusione (`represented`) solo con un segnale forte:**
| Segnale | Stage | Come |
|---|---|---|
| Handle nel roster di un'agenzia | prefilter | scrape periodico (mensile) delle pagine roster delle agenzie talent italiane → tabella `agency_roster`. La lista delle agenzie si costruisce con una ricerca iniziale e si aggiorna a mano |
| Email business su dominio di agenzia | agency_check | tabella `domains` (dominio → classificazione). Dominio nuovo → crawl della homepage → LLM: "talent/influencer agency sì/no". Risultato in cache per sempre |
| Bio con management + contatto o account esterno | prefilter | regex IT/EN (`management`, `mgmt`, `managed by`, `rep by`, `booking`, `talent`, `agenzia`) seguita da email o @menzione non del creator |
| Account menzionato in bio è un'agenzia | agency_check | scrape del profilo menzionato: categoria IG "Talent agent"/agenzia o bio che parla di roster/talent |
| Link in bio verso management o form di agenzia | agency_check | crawler su Linktree/sito: link "Management", "Booking", form su dominio di agenzia |
| Il creator lo dichiara | CRM | colonna risposta → `represented`, esce dalla lista |

**`independent`**: email business su gmail/icloud/outlook/libero/hotmail, oppure su un dominio che corrisponde al nome o al sito del creator.
**`unknown`**: nessun segnale. Resta nei risultati, marcato "da verificare"; la bozza del primo messaggio include la domanda sul management.
**Mai motivo di esclusione**: molte sponsorizzazioni, grandi brand, "per collaborazioni scrivi a" con email personale.
**Ricontrollo**: a ogni nuovo snapshot, se cambiano bio, email o link esterno, si ripete agency_check prima dell'outreach.

## 6. Enrich + metriche (codice, niente LLM)
Enrich: profilo (bio, follower, seguiti, categoria, email business, link esterno, verificato) + ultimi 40 post/reel (like, commenti, view, data, caption, tag, menzioni, flag sponsorizzazione) + crawler del link in bio.

Metriche:
- **ER mediano** (like+commenti)/follower sugli ultimi 40 post;
- **view mediane dei reel / follower**: l'indicatore più importante;
- frequenza (post a settimana), giorni dall'ultimo post;
- **sponsorizzati vs organici**: rapporto tra ER dei post `#adv` e ER degli organici (sotto ~0,5 = l'audience ignora le sponsorizzazioni);
- quota di post sponsorizzati (troppi = audience satura);
- crescita follower 30 giorni (da `snapshots`, disponibile dal secondo run);
- **solo per la shortlist** (scrape commenti, ~200 commenti per creator): quota di commenti solo emoji o generici, commentatori ripetuti tra post (indizio di pod), quota di commenti in italiano (proxy dell'audience italiana).

## 7. Classify (LLM su bio + caption)
Una chiamata per creator con output JSON e citazione obbligatoria (ogni campo con `post_id` + estratto; un estratto che non è substring dell'input viene scartato):
- `niche_fit` 0–3 (0 = fuori tema, 3 = alimentazione e tracciamento sono il cuore del profilo);
- `topics` (ricette, meal prep, ricomposizione, dimagrimento, sport, lifestyle…);
- `brand_safety` ok/rischio + motivo (diete estreme, claim medici, integratori aggressivi, contenuti off-brand);
- `competitor_promos` [{brand, post_id, data}];
- `accepts_collabs` (ha già fatto collaborazioni con codice o `#adv`);
- `format_fit` (fa reel con il telefono in mano / racconta la sua giornata alimentare: formato naturale per mostrare un'app).

Modello piccolo ed economico; prompt e versione in file; versione salvata in `scores`.

## 8. Score
Calcolato dal codice, pesi in `scoring.yaml`. Proposta iniziale (100 punti):
| Componente | Peso |
|---|---|
| niche_fit | 30 |
| view reel / follower | 20 |
| ER mediano (normalizzato per fascia follower) | 10 |
| ha promosso un concorrente (bonus scalato per recenza; >6 mesi pieno, <2 mesi dimezzato per rischio esclusiva) | 15 |
| accepts_collabs | 10 |
| format_fit | 10 |
| qualità commenti (solo shortlist) | 5 |
| Penalità: brand_safety rischio | escluso |
| Penalità: sponsorizzati vs organici < 0,5 | −10 |
| Penalità: quota sponsorizzati > 40% | −10 |

Shortlist = top N per score con `agency_status ≠ represented` e `brand_safety = ok`. A parità di score, gli `independent` passano prima degli `unknown`.

## 9. CRM = Google Sheet
30 attivi e ~200 contatti al mese stanno in uno Sheet. Colonne script (A–P) e colonne umane (Q+) separate, upsert per `creator_id`.
- Script: handle, link, follower, score, fascia, `agenzia` (indipendente + fonte / da verificare), metriche chiave, niche e topics, concorrenti promossi, post di riferimento per l'aggancio, contatto (email/DM), bozza del primo messaggio.
- Umano: stato (da contattare / contattato / risposto / call / attivo / rifiutato / ha agenzia), date, note, codice assegnato.
- Tab: `Shortlist`, `In corso`, `Attivi`, `Run log` (volumi per stage, rendimento per fonte).

## 10. Outreach
- Contatto via DM dall'account Instagram di Olivia, oppure via email se è business. **Invio sempre manuale**, 5–8 al giorno: i DM automatici fanno bannare l'account.
- Bozza generata dall'LLM: cita un contenuto reale del creator, 3–4 righe, niente link nel primo messaggio, proposta chiara (codice sconto per la sua audience + commissione per ogni abbonamento). Agli `unknown` aggiunge la domanda "gestisci tu le collaborazioni o hai un management?".
- Un solo follow-up dopo 5–7 giorni.

## 11. Codice creator e tracking
Su mobile l'attribuzione link → store è debole (iOS), quindi il **codice è la fonte di verità**.
- Ogni creator ha un codice univoco (es. `MARTA10`) che dà uno sconto o una prova più lunga all'utente e attribuisce l'abbonamento al creator.
- Il codice si inserisce in **onboarding o paywall**. Verificare cosa esiste già nel backend e nel sistema di pagamento prima di costruire: serve `creator → codice → riscatti → abbonamenti → rinnovi/rimborsi`.
- Report mensile per creator: riscatti, trial, abbonati paganti, revenue netta, retention a 30/60 giorni, commissione dovuta.
- Opzionale: link tracciato con UTM verso una landing che mostra il codice (segnale secondario, non per il calcolo della commissione).
- Le performance rientrano nella pipeline: i creator che convertono diventano seed per l'espansione e calibrano i pesi dello score.

## 12. Dati
SQLite (sotto i 10k creator basta), tabelle:
`creators`, `source_hits`, `source_cursors`, `posts`, `comments_sample`, `snapshots`, `agency_roster`, `domains`, `scores` (con versione pesi/prompt), `outreach`, `codes`, `conversions` (import dal backend), `runs`.

Minimizzazione GDPR: gli esclusi restano solo con handle, motivo e data (servono per non riscaricarli) e vengono cancellati dopo 6 mesi; post e commenti dei creator non in shortlist si cancellano dopo 30 giorni. Informativa al primo contatto (art. 14) e legittimo interesse documentato. Da verificare con un legale.

## 13. Esecuzione
- CLI Python: `gtm discover|prefilter|enrich|agency|metrics|classify|score|sync|report --limit N --dry-run`.
- Primo mese: lanciato a mano, settimanale. Poi cron (GitHub Actions o simile).
- Costi: Apify fa pagare a risultato. L'imbuto (discovery larga → enrich solo sui pre-filtrati → commenti solo sulla shortlist) tiene il costo mensile basso; i prezzi dei singoli actor vanno verificati sullo store Apify con il run di prova.

## 14. Contratto e brief per il creator (fuori dalla pipeline, ma bloccanti)
- Accordo semplice: codice, % di commissione, durata, pagamento mensile, nessuna esclusiva richiesta.
- Obbligo di segnalare la pubblicità (`#adv`, linee guida AGCOM).
- Niente claim medici o di dimagrimento garantito.
- Forma di pagamento per creator senza P.IVA (prestazione occasionale e simili): da definire con il commercialista.

## 15. Prossimi passi
1. Run di prova Apify su 3 hashtag + 2 concorrenti → 50 profili: verificare campi disponibili (email business, related, flag sponsorizzazione) e costo reale.
2. Lista agenzie talent italiane + scraper roster.
3. 30 profili etichettati a mano (fit sì/no, agenzia sì/no) per tarare pre-filtri, prompt e score.
4. Verificare il supporto ai codici creator nel backend e nel paywall.
5. Implementazione stage per stage, poi primo batch di 50 contatti.
