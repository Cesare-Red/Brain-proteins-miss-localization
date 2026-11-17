# NeuroLoc-Δ · Risposta alla verifica del 16 novembre 2026

Per ogni punto di `VERIFICA_METODOLOGIA.md`: cosa è stato cambiato e cosa resta da fare. Le modifiche sono state
provate su dati di prova (e su file veri dove possibile); **la pipeline completa non è stata rieseguita**: i numeri
nuovi escono dalla prossima esecuzione 01 → 02 → 03.

Legenda: **Risolto** = corretto nel codice, il risultato si vede rilanciando · **Tocca a te** = serve un'azione fuori
dai notebook · **Dichiarato** = limite reale, scritto nei notebook e da scrivere nell'articolo.

## A · Problemi gravi

**A1 · Confronto con DeepLoc 2.0 — Risolto in locale, da eseguire.** Il server non si può usare: DeepLoc 2.0 e il
ProtT5-MHA di Wefers sono **rifatti dal loro codice** nello 02 (sezione 8d) e addestrati sugli stessi dati, partizioni e
test della nostra rete; la nostra metodologia sugli stessi dati è lo stadio `abl_solo_deeploc`. Confronto appaiato sul
test HPA (02, sezione 10) e sul test HOU fuori fold (03, sezione 12). Per avere una predizione fuori fold di ogni metodo
su ogni proteina HOU, i modelli addestrati sulle sole etichette curate ora predicono tutta la partizione di
valutazione (cella delle reti, 02). I valori pubblicati di DeepLoc 2.0 restano **fuori** dalla tabella dei modelli.

**A2 · Test HPA — Risolto.**
- Collegamento **per gene** (ENSP → accession → ENSG → voce Swiss-Prot dello stesso gene), dopo sequenza identica ed
  ENSP diretto; i 5-meri restano come riserva e come controllo incrociato. L'avviso scatta sopra il 5% di non collegate.
- Una isoforma per gene: quella **identica alla canonica** se c'è, altrimenti la più lunga.
- Per ogni proteina del test si registra se è identica alla canonica e quanta parte ne copre; lo 02 riporta l'AUC
  per strato (identica, almeno metà, frammento).
- Dichiarato: il test è un sottoinsieme del benchmark, misura 6 classi su 9, e le sequenze-frammento deprimono i
  valori assoluti (non i confronti appaiati).

**A3 · Lysosome/Vacuole — Risolto.** Scelta la definizione di DeepLoc: **gli endosomi sono nella classe**. Allineati
UniProt, GO, HPA ("Endosomes"), marcatori LOPIT (endosomi precoci) e BioID. Per le "vescicole" generiche (UniProt, GO,
HPA "Vesicles") la classe resta "non osservata" invece di diventare un'assenza. Lo 01 stampa la sensibilità di ogni
fonte contro le etichette curate: deve salire rispetto a 0,54 / 0,28 / 0,02.

**A4 · Architettura — Risolto nel metodo, dichiarato nel risultato.**
- Bootstrap appaiato anche sull'**AUPRC** macro, con le differenze e i loro intervalli.
- Varianti, prove sui dati e riferimenti con **3 semi** (prima 1): tutti i modelli a confronto sono la media di 15
  reti. Tabella "una rete per seme" con media e deviazione standard.
- AUPRC in CV anche sulle sole 6 classi del test.
- **Regola di scelta**: il più semplice tra gli equivalenti (entro 2 deviazioni standard); ordine di semplicità
  scritto prima dei risultati. Le prove sui dati non sono candidate.
- Ogni contributo (modello, dati, fonti HPA, attenzione sui residui, scheda) è stampato con AUC e AUPRC e intervallo.
- Dichiarato: se i numeri restano quelli del 16 novembre, l'architettura **non** è un contributo; lo sono i dati.
- Costo: circa 20 addestramenti in più nello 02.

**A5 · Tabella finale — Risolto.** Predizione e annotazioni sono **separate**:
- organelli: "% probabilità" = predizione (sequenza + partner, tarata, fuori fold), senza le annotazioni della proteina;
- compartimenti neuronali: **percentile** tra le proteine neuronali, non una probabilità; mielina vuota;
- colonne "localizzazioni annotate", "localizzazioni predette" (per i neuronali: candidate nel 5% più alto) e la loro unione;
- la fusione con le misure resta un'analisi (sezione 4) e non entra nella tabella; la sua verifica ora guarda anche i
  soli positivi curati;
- lo 03 stampa quante annotazioni la predizione ritrova e quante candidate sono nuove
  (`data/tabella_annotate_e_predette.xlsx`). Pagina HTML e casi di studio aggiornati.
- Dichiarato: il valore aggiunto dei compartimenti neuronali resta non dimostrato (nessun test esterno).

## B · Problemi di gravità media

**B1 · Varianti — Risolto.** Bootstrap **per proteina**; correzione di Benjamini-Hochberg sui 13 segnali; minimo di
30 varianti informative; i segnali che contengono l'LLR (`rischio_ritenzione_RE`, `rischio_ripiegamento`,
`score_misloc`) non contano come "spostamento". Un solo protocollo per tutte le varianti (modello che non ha visto la
proteina). Il verdetto stampa anche score_variante contro LLR entro la stessa proteina. Testo allineato all'output;
dichiarati l'LLR "wild-type marginal" in fp16 e la composizione di `score_variante`.

**B2 · CV ottimistica, righe duplicate — Risolto / dichiarato.** Tolte "Rete, pool esteso" e "Rete, 1 seme"; "Rete
neurale + k-NN" compare solo se α > 0. L'ottimismo dell'AUPRC in CV è stampato e scritto.

**B3 · Omologia tra candidati — Tocca a te (facoltativo).** Lo 02 scrive `data/omologia_test.fasta` e
`data/omologia_training.fasta`. Con MMseqs2 installato:
`mmseqs easy-search data/omologia_test.fasta data/omologia_training.fasta data/esterni/mmseqs_test_vs_training.m8 tmp --min-seq-id 0.3 -c 0.5 --cov-mode 2 -s 7.5`
Rilanciando la sezione 4 lo 02 dice quante omologhe erano sfuggite.

**B4 · Calibrazione — Dichiarato.** Tarata sulla frequenza di annotazione, non di presenza (03, sezione 3).

**B5 · Ripartizione % — Dichiarato.** Solo citosol e nucleo sono misurati; il resto è un'euristica (notebook e pagina).

**B6 · Wefers — Risolto.** Il test HOU è ora un confronto **appaiato**: stesse proteine, stessi ricampionamenti, quattro
metodi (NeuroLoc-Δ sugli stessi dati, DeepLoc 2.0 e Wefers rifatti, NeuroLoc-Δ completo), anche sulle sole proteine
HOU fuori da DeepLoc. Resta l'AUC del citoplasma contro "cytosol o cytoskeleton". Il sistema completo sul test HOU non è
indipendente (etichette dalle stesse banche dati): la sua riga è contesto. I valori pubblicati restano come contesto.

**B7.** Malattie: dichiarata la circolarità delle encefalopatie energetiche. Perimetro: soglia in nCPM separata da
quella in nTPM; le proteine con un'annotazione neuronale restano neuronali anche sotto soglia. Scheda: tolto il simbolo
del gene. Pesi dei positivi: corretta l'intestazione dello 01 (valgono solo per le etichette curate).

## C · Codice e riproducibilità
- **Esecuzione pulita e commit — Tocca a te**: Restart & Run All di 01, 02, 03, poi commit.
- README aggiornato. Testo corrente corretto (`abl_senza_card`, "0,81–0,85", commenti su GTX 1080 e "bit per bit").
- PCA degli alberi e dei residui stimate senza il test; arresto anticipato degli alberi su una partizione di omologia.
- `assert` sugli embedding completi prima di procedere.
- ClinVar: tolte le sostituzioni con classi contraddittorie; quota leave-one-out dopo la deduplicazione.
- Proteine nuove (6b): stessa lista di parole dello 01, senza Keywords e senza simbolo del gene; stesse colonne della
  tabella. Resta mai eseguita.
- Versioni: lo 01 scrive `requirements.lock.txt` da solo. Dichiarato: la versione di HPA non è nel file scaricato
  (resta data e SHA-256); il test di DeepLoc viene da HPA del 2022, le etichette di training da HPA attuale.

## Non fatto
- Nessuna riesecuzione; riallineamento della bozza (le tre frasi segnalate vanno cambiate, e "DeepLoc 2.0" va
  scritto "DeepLoc 2.0 riaddestrato sui nostri dati").
- Le due repliche sono provate su dati sintetici (si addestrano e predicono), non ancora sui dati veri.
- Non verificati: JavaScript della pagina, hook sui layer intermedi, indipendenza dei marcatori LOPIT.
