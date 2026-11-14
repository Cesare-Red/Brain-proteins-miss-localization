# NeuroLoc-Δ · Risposta alla revisione del 14 novembre 2026

Per ogni punto di `STATO_CODICE.md`: cosa dice la critica, come è stata affrontata, dove sta la modifica, cosa resta.

Le modifiche sono state provate su dati di prova e, dove possibile, su file veri (i tre esperimenti LOPIT, i marcatori,
il test HOU). **La pipeline completa non è stata rieseguita**: i numeri nuovi escono dalla prossima esecuzione
01 → 02 → 03. Dove un punto dipende da quell'esecuzione, o da un passo manuale, è scritto.

Legenda: **Risolto** = corretto nel codice · **Da eseguire** = corretto, ma il risultato si vede solo rilanciando ·
**Passo manuale** = serve un'azione fuori dai notebook · **Dichiarato** = è un limite reale: non si elimina, si scrive.

## Problemi gravi

### G1 · Test HPA ridotto a 292 proteine su 1.717 — Risolto, da eseguire
- **Causa confermata.** Gli ID del test sono ENSP (proteina Ensembl); il codice cercava ENSG e simboli genici.
- **Correzione (01, cella "Proteine").** Tre vie, dalla più sicura: sequenza identica; ENSP → UniProt con la tabella
  ufficiale `HUMAN_9606_idmapping.dat.gz` (nuovo download); riserva per gli ENSP ritirati: la voce UniProt che contiene
  almeno metà dei 5-meri della sequenza del test. Il notebook stampa quante proteine passano per ogni via e la
  concordanza tra ENSP e 5-meri dove ci sono entrambe (controllo incrociato), e avvisa se più del 10% resta scollegato.
- **Scelte.** La proteina del test tiene la **sua** sequenza (è il benchmark pubblicato, la stessa che riceve il server
  di DeepLoc 2.0); la voce UniProt serve per gene, perimetro e struttura. Escono le proteine il cui gene è già nel
  training di DeepLoc. La voce UniProt di ogni proteina del test esce dalle proteine aggiunte; un `assert` lo verifica.
- **Resta.** Extracellulare, lisosoma e perossisoma non sono misurati da HPA: su quelle tre classi il test non potrà
  mai dire nulla. Il notebook stampa i positivi per classe del nuovo test.

### G2 · Confronto con lo stato dell'arte non eseguito — Passo manuale + risolto nella presentazione
- **Predizioni ufficiali.** Non si possono generare da qui (il server DTU va usato a mano). Lo 02 ora scrive i FASTA
  con l'ID del test come intestazione (prima il numero di riga: i CSV smettevano di corrispondere a ogni nuovo 01),
  stampa quanta parte del test è coperta dai CSV e, se mancano, scrive in chiaro **"Confronto con DeepLoc 2.0: NON
  disponibile"**. Con i CSV, il confronto è appaiato (stesse proteine, bootstrap della differenza).
- **Il riferimento "riprodotto" non è DeepLoc 2.0.** Rinominato ovunque (tabelle, figure, testo) in *riferimento
  semplice (ProtT5 medio + MLP; non è DeepLoc 2.0)*. Ha ora una sua piccola ricerca di iperparametri (8
  configurazioni, stadio `rif_screening`): prima li ereditava dalla rete vincente.
- **Valori pubblicati.** Restano in tabella con l'etichetta "test intero, 1.717 proteine: non confrontabile riga per riga".
- **Wefers.** "Dentro l'intervallo" è scritto come parità, non superiorità.
- **Resta.** Riprodurre davvero DeepLoc 2.0 (attenzione sui residui, 28.000 proteine eucariote) è fuori dal perimetro
  umano del progetto: il confronto vero è con le predizioni ufficiali.

### G3 · Le varianti non predicono la mislocalizzazione — Dichiarato (risultato negativo)
- Non c'è un modo onesto di "aggiustarlo": è un risultato. Lo 03 lo presenta come tale.
- **Sezione 7 riscritta**: `score_variante` è una *priorità per patogenicità*; i segnali di spostamento sono ipotesi.
- **Intervalli di confidenza** (bootstrap, 1.000 ricampionamenti) per ogni segnale, su ClinVar e sulle varianti
  misurate, con la colonna "distingue (IC esclude 0,5)". Il notebook stampa il verdetto dai numeri: quanti segnali di
  spostamento hanno un IC sopra 0,5, accanto all'LLR di ESM-2.
- Uscita: `data/varianti_valutazione.xlsx`.

### G4 · La scheda testuale porta la risposta — Risolto, da eseguire
- **Pulizia per frasi e per voci (01).** Esce l'intera frase (testo libero) o voce (liste GO, famiglia, complessi,
  domini) che nomina un compartimento: niente più frammenti ("trans", "network", "helix"). Le **Keywords** non entrano
  più. Corretto anche un difetto della lista: `import\w*` cancellava "important".
- **Modello principale di sola sequenza (02).** La scheda è fuori dalla griglia, dalla logistica e dagli alberi. Entra
  solo nella variante `con_scheda`, chiamata "aumentata dalla conoscenza dei curatori" ed esclusa dalla scelta del
  modello. Lo 02 stampa il suo contributo sul test con l'intervallo.
- **Resta (dichiarato).** Una scheda scritta dai curatori non può essere "senza localizzazione": la funzione è
  correlata al luogo. La sezione 4b continua a misurarlo.

## Problemi di gravità media

### M1 · Scelta del modello — Risolto, da eseguire
- **Regola unica**: la migliore AUPRC in CV tra i modelli di sola sequenza. Le configurazioni semplici (solo medie)
  sono nella griglia e tra i candidati: se vincono, vincono. Niente più "ablazione" che batte il modello scelto.
- **Rumore dichiarato**: deviazione standard tra semi e lista dei modelli entro 2 DS dal migliore ("equivalenti").
- **Ensemble annidato**: per il fold k la combinazione si sceglie sugli altri quattro. Il notebook stampa la stima
  annidata (da riportare) e quella ottimistica. Nessuna "ablazione" tra i membri.
- **"Guadagno del modello"**: la frase è sostituita da contributi separati, ognuno con IC: dati in più (sul riferimento
  e sulla rete), fonti HPA, scheda.
- **NeuroLoc-LA, struttura, ProtT5, attenzione**: il testo ora dice che nell'ultima esecuzione erano entro il rumore o
  peggiori. Sono risultati da riportare così.

### M2 · Compartimenti neuronali — Dichiarato + risolto dove possibile
- **Mielina**: sotto 30 positivi una classe non si valuta. È tolta da tabelle, curve e controlli (resta l'uscita, marcata).
- **Positivi e non etichettati**: scritto nello 02 e nello 03; le probabilità neuronali sono un ordinamento.
- **Ancora del canale**: la specificità è fissata a 1 per le annotazioni sperimentali (SynGO, UniProt sperimentale, GO
  sperimentale) su tutte le classi neuronali. La scala resta non identificabile: è dichiarato.
- **Omologia tra proteine aggiunte**: ora si allineano anche aggiunta–aggiunta; le famiglie (componenti connesse)
  stanno nello stesso fold. Un'aggiunta con un'omologa curata in un altro fold esce dal training e dalle valutazioni
  fuori fold (anche nel contesto HOU). Gli allineamenti passano da circa 130.000 a circa 200.000 (30–35 minuti).
- **Resta.** Nessun test esterno per le classi neuronali; la ricerca degli omologhi è tra candidati, non tutti contro tutti.

### M3 · Fusione con le misure — Risolto, da eseguire
- **Fonti dipendenti**: LOPIT ×3, HPA ×2 e SynGO ×2 contano una volta (media dei LLR nel gruppo).
- **Sensibilità gonfiate**: per gli organelli si usano quelle osservate contro DeepLoc, stimate fuori fold.
- **Saturazione e calibrazione**: una logistica per classe impara quanto pesare prior e misure; ECE, quota di
  probabilità sotto 1% o sopra 99% ed "errori sicuri" sono misurati **dopo** la fusione, accanto alla fusione vecchia
  (`data/verifica_fusione.xlsx`).
- **Malattie**: l'analisi usa le probabilità della sola sequenza, fuori fold.
- **Resta.** Per le classi neuronali il peso delle misure è preso dagli organelli: le loro probabilità fuse non sono verificate.

### M4 · Altri punti
- **Copertura conformale** — Risolto: non si chiama più così. È "insieme ad alta sensibilità", con il richiamo misurato sul test.
- **Perimetro "cervello"** — Dichiarato in testa allo 01 e nel README (80% di Swiss-Prot; etichette da linee cellulari).
- **Omologia test–training** — Risolto: AUC per strato (con e senza omologa) in `scelta_metodologia.xlsx`.
- **Fonte del test** — Risolto: il contributo delle fonti HPA è stampato con IC e la frase che va dichiarata.
- **ClinVar** — Risolto: quota di patogeniche in leave-one-out e mai tra le feature; `score_variante` è il punteggio
  principale; AUROC entro proteina anche su ClinVar; controllo della numerazione per gene (≥ 90% di residui
  coincidenti); il campione di 4.000 è dichiarato. Resta: il collegamento non è per trascritto.
- **LOPIT** — Risolto: classe "altro" per i marcatori fuori schema (classi aperte); le altre classi di una proteina
  sono assenze solo sotto probabilità 0,05, altrimenti "non osservate".

## Codice e riproducibilità
- **Esecuzione non pulita** — I notebook sono consegnati senza output. Da fare: Restart & Run All.
- **Cella morta 2d (02)** — Tolti download e calcolo; restano le funzioni usate dalla 3a. *È l'unica modifica di codice
  nelle sezioni 0–2.*
- **Markdown non aggiornato** — In testa allo 02 c'è "Stato attuale"; le sezioni storiche sono dichiarate diario.
- **Cache degli embedding** — Legata a un'impronta SHA-256 (nuova sezione 1b). Alla prima esecuzione la cache
  esistente viene spostata, non cancellata, in `data/cache_vecchia/`.
- **Versioni** — `requirements.txt` fissa torch, transformers, pandas e scikit-learn (< 1.11). `SVC(probability=True)`
  resta: il sostituto è stato provato sui dati veri ed è peggiore (marcatori giusti da 0,94 a 0,90; nucleo perso in
  dLOPIT 2024).
- **Dati esterni** — `data/versioni_dati.csv` e `cache/registro_download.tsv`.
- **Sottoinsieme HOU duplicato** — Tolto.
- **Commit** — Da fare: questa sessione non ha accesso a GitHub.

## Cosa non è stato fatto
- Nessuna esecuzione completa: embedding e rete richiedono GPU e i dati scaricati.
- Le predizioni ufficiali di DeepLoc 2.0.
- La bozza dell'articolo non è stata riallineata: va fatto dopo l'esecuzione di riferimento.
- Il percorso delle proteine nuove (03, sezione 6b) resta non provato; la sua scheda usa ancora la pulizia per parole,
  che non conta più per il modello principale (di sola sequenza).
