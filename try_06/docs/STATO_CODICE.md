# NeuroLoc-Δ (try_05) · Stato del codice e della metodologia

Revisione del 14 novembre 2026 sui tre notebook così come sono stati eseguiti (codice + output salvati), più alcune
verifiche dirette sui file in `data/` e `cache/`. Nessun notebook è stato rieseguito e nessun file del progetto è stato
modificato. La bozza `docs/NeuroLoc_articolo_revisione.docx` non è ancora stata confrontata con questi risultati.

## In sintesi

La pipeline gira dall'inizio alla fine e l'impianto di validazione è in buona parte corretto (vedi "Cosa è a posto").
Ci sono però quattro problemi che, così com'è, impediscono di sostenere in un articolo le affermazioni principali:

1. il test indipendente è ridotto a 292 proteine su 1.717 per un errore nel collegamento degli identificativi;
2. il confronto con lo stato dell'arte non c'è: le predizioni ufficiali di DeepLoc 2.0 non sono state caricate e la
   "ricetta riprodotta" non è DeepLoc 2.0;
3. i segnali di *mis*-localizzazione delle varianti non predicono la mislocalizzazione misurata (AUROC ≈ 0,5);
4. la scheda testuale contiene ancora indizi di localizzazione e da sola vale quanto la sequenza.

Seguono problemi di gravità media (scelta del modello, classi neuronali, fusione con le misure) e note sul codice.

## Problemi gravi

### G1 · Test HPA: persi 1.398 casi su 1.717 (01, cella "Proteine")
- `deeploc_hpa_test.csv` identifica le proteine con ID **ENSP** (es. `ENSP00000355743`); il codice cerca `ENSG\d+`
  e poi il simbolo genico. Nessuno dei due può combaciare: l'output dice "per gene 0, non collegate 1,398".
- Restano solo le 319 proteine la cui sequenza Ensembl è identica alla canonica UniProt, 292 dopo il filtro cervello.
  È il 17% del test e non è un campione casuale.
- Positivi per classe nel test: nucleo 163, citoplasma 85, membrana 37, mitocondrio 37, **Golgi 16, RE 12**.
  Extracellulare, lisosoma e perossisoma non hanno alcuna valutazione sul test. MCC e AUC di RE e Golgi sono instabili.
- Correzione: mappare ENSP → UniProt (ID mapping di UniProt o BioMart), oppure tenere le sequenze del test così come
  sono. Poi vanno rifatti 02 e 03.

### G2 · Confronto con lo stato dell'arte non eseguito (02, sezione 9)
- Output: "Predizioni ufficiali di DeepLoc 2.0 trovate: no". In `data/esterni/` non c'è nessun `deeploc2_*.csv`.
- Lo "stato dell'arte riprodotto" è la media di ProtT5 + percettrone, addestrato su 6.371 proteine umane, con gli
  iperparametri ereditati dalla rete vincente e non ottimizzati. DeepLoc 2.0 usa attention pooling sui residui e
  ~28.000 proteine eucariote: il +0,046 di AUC è contro un riferimento più debole dell'originale.
- I valori pubblicati di DeepLoc 2.0 nella tabella si riferiscono al test intero (1.717), non alle 292 proteine.
- L'unico confronto esterno disponibile è il contesto Wefers (03, sezione 12), non appaiato: sola sequenza AUC macro
  0,906 (IC 0,899–0,914) contro 0,903 di DeepLoc2-ProtT5 e 0,905 di LAProtT5. È parità, non superiorità.

### G3 · Varianti: la variazione di localizzazione non predice la mislocalizzazione (03, sezione 7)
Sulle 247 varianti con localizzazione misurata (Lacoste, 132 mislocalizzate / 115 no):

| segnale | AUROC |
|---|---|
| `score_misloc` (non supervisionato) | 0,513 |
| JSD tra profili WT e mutante | 0,517 |
| variazione massima dei meccanismi | 0,448 |
| rischio nucleo-citoplasma | 0,439 |
| rischio sinaptico | 0,525 |
| LLR di ESM-2 (da solo) | 0,603 |
| punteggio supervisionato | 0,590 |

- I segnali specifici di localizzazione sono a livello del caso. L'unico segnale è l'LLR, che misura il danno
  generico della sostituzione, non lo spostamento.
- Su ClinVar il supervisionato arriva a 0,929, ma l'LLR da solo fa 0,902 e la JSD 0,517: il contributo della parte
  di localizzazione è nullo o quasi.
- Con 247 varianti l'incertezza è circa ±0,07; non sono riportati intervalli di confidenza.
- Per l'articolo: va presentato come risultato negativo, oppure la parte sulle varianti va ridimensionata a
  "priorità per patogenicità".

### G4 · La scheda testuale porta ancora la risposta (01 cella card, 02 sezioni 3c e 4b)
- La pulizia cancella le parole, non le frasi: restano i frammenti. L'analisi χ² dello stesso notebook li mostra come
  parole più associate alle classi: "apparatus", "trans", "tgn", "network" (Golgi); "helix", "cell" (membrana, da
  *transmembrane helix* e *cell membrane*); "late", "multivesicular" (lisosoma); "mt-rrna", "mt-lsu", "mrp-"
  (mitocondrio); "fluids" (extracellulare).
- La scheda da sola (logistica) fa 0,599 di AUPRC macro contro 0,593 degli embedding di sequenza.
- Scheda e etichette DeepLoc vengono dagli stessi curatori UniProt: in CV il guadagno (+0,045 AUPRC) è in parte
  circolare. Sul test HPA il guadagno è +0,008 di AUC (0,869 contro 0,861), dentro il rumore.
- Per l'articolo: il modello principale nei confronti deve essere quello di sola sequenza (`abl_senza_card`); la
  versione con scheda va presentata come "aumentata dalla conoscenza" e la pulizia va rifatta togliendo campi e
  frasi interi (in particolare le Keywords).

## Problemi di gravità media

### M1 · La scelta del modello contraddice la regola dichiarata (02, sezioni 7–8)
- L'ablazione "senza pezzi" (solo medie di ESM-2 e ProtT5) batte la configurazione scelta: AUPRC CV 0,716 contro
  0,696, AUC test 0,876 contro 0,869. Il rumore tra semi è 0,695–0,700.
- Struttura AlphaFold, ProtT5, descrittori e motivi, attenzione tra token: tutti entro il rumore. NeuroLoc-LA
  (attenzione sui residui) è peggiore della rete semplice: 0,688.
- Gli unici ingredienti con un effetto misurabile sono il pool esteso con più fonti (+0,02–0,04) e la scheda.
- L'ensemble finale è scelto tra 63 combinazioni sulle stesse predizioni out-of-fold su cui poi si riporta la
  metrica (0,729, ottimistica) e contiene l'"ablazione" come membro.
- La frase "a parità di dati … è il guadagno del modello" attribuisce al modello anche il contributo della scheda;
  inoltre la ricetta semplice con più dati non migliora (0,828 contro 0,827).

### M2 · Compartimenti neuronali: nessuna valutazione indipendente (02 sezione 4, 03 sezioni 2 e 10)
- Nessuna etichetta curata e nessun test esterno: il riferimento sono le stesse fonti usate per l'addestramento.
- I negativi sono "non annotato" (regola delle assenze): è un problema positivi/non etichettati, e AUPRC e
  calibrazione vanno lette di conseguenza.
- **Mielina: 7 positivi** (AUPRC 0,006). Non è una classe valutabile e va tolta o dichiarata tale. Dendrite: AUPRC 0,148.
- Per assone, dendrite e mielina il canale di rumore non ha alcuna ancora: sensibilità e specificità non sono
  identificabili.
- Le partizioni delle proteine aggiunte separano l'omologia verso DeepLoc, non quella tra proteine aggiunte. Stima
  per difetto con i 5-meri condivisi: 182 proteine aggiunte hanno un quasi-duplicato in un altro fold (172 con
  un'altra aggiunta). Gli omologhi lontani non sono misurati. Riguarda tutte le metriche out-of-fold sul pool esteso
  (03 sezione 2, ROC neuronali, contesto HOU). Sulle proteine DeepLoc l'effetto è trascurabile (25 su 6.371).

### M3 · Fusione con le misure e tabella finale (03, sezioni 4, 6, 8)
- La somma dei rapporti di verosimiglianza assume fonti indipendenti. Non lo sono: tre mappe LOPIT con gli stessi
  marcatori e lo stesso classificatore, due livelli HPA, e `uniprot_forte` che coincide quasi con DeepLoc.
- Le sensibilità stimate dal canale sono più alte di quelle osservate (HPA Approved 0,46 contro 0,18; UniProt debole
  0,70 contro 0,29): le assenze pesano troppo.
- Risultato: probabilità saturate a 0 o 100. Esempio: FUS citoplasma 0,0%, benché una mappa LOPIT la dia nel
  citoplasma e la proteina faccia la spola.
- La calibrazione mostrata (ECE 0,003–0,02) è misurata **prima** della fusione; le "% probabilità" pubblicate nella
  tabella e nella pagina HTML sono quelle **dopo**, mai verificate.
- L'analisi delle malattie usa le probabilità fuse: per le proteine annotate ripete le annotazioni. Delle 11
  associazioni attese ne ritrova 4, e 4 sono nel verso opposto (riportato onestamente nel notebook).

### M4 · Altri punti
- **Copertura conformale**: 0,81–0,85 sul test contro il 0,90 nominale, per tutti i modelli. Così non si può
  chiamare "insieme conformale al 90%".
- **Perimetro "cervello"**: passa l'80% delle proteine umane Swiss-Prot (soglia 1 nTPM) e le etichette vengono da
  HEK293T, HeLa e U2OS. La specificità cerebrale va dichiarata per quello che è.
- **Omologia test–training**: almeno il 16,8% delle proteine del test ha un'omologa ≥ 30% nel training DeepLoc.
  Servono metriche stratificate.
- **Fonte del test**: senza le fonti HPA l'AUC sul test scende da 0,869 a 0,858. Una parte del guadagno è
  allineamento alla fonte delle etichette.
- **ClinVar**: `score_supervisionato` usa anche il contesto genico, che include la quota di patogeniche del gene
  calcolata con la variante stessa. Va riportato solo `score_variante` (0,926). Manca l'AUROC entro gene su ClinVar.
  Le varianti sono collegate per GeneID e residuo WT, non per trascritto. Il campione è di 4.000 su ~175.000.
- **LOPIT**: classificatore a classi chiuse (i marcatori di endosomi e simili sono scartati, le loro proteine
  finiscono forzate in un'altra classe) e una sola etichetta per proteina, che la regola delle assenze trasforma in
  negativi per le altre.

## Note sul codice e sulla riproducibilità
- **Esecuzione non pulita**: lo 01 parte dal contatore 19 e salta da 30 a 32; lo 02 salta da 7 a 15. Prima
  dell'articolo serve un "Restart & Run All" per ciascun notebook.
- **Cella morta** (02, 2d): scarica AlphaFold `model_v4`, che non esiste più, scrive 15.537 file vuoti e ottiene 0
  strutture; la cella 3a rimedia con la v6. La 2d va tolta.
- **Markdown non aggiornato**: GTX 1080, "~22.000 proteine", ESM-2 150M, 287 proteine di test, pesi delle specie non
  umane, MitoCarta "in `cache/`". Descrivono versioni precedenti.
- **Cache degli embedding**: riusata in base al solo numero di righe. In questa esecuzione è coerente (verificato:
  date dei file successive a `proteine.parquet`, coseno mutante–WT ≈ 1,000 su 400 varianti, righe per-residuo uguali
  alla somma delle lunghezze). Conviene legarla a un hash dell'elenco delle sequenze.
- **Versioni**: `requirements.txt` ha solo limiti inferiori. Ambiente usato: Python 3.14.4, torch 2.15, scikit-learn
  1.9.1, pandas 3.1.1, transformers 5.22. `SVC(probability=True)` sparisce in scikit-learn 1.11.
- **Dati esterni**: UniProt, HPA, ClinVar, GOA scaricati come "ultima versione", senza registrare release e data.
- **Residui**: nello 03 il sottoinsieme "HOU nel perimetro cerebrale" è identico all'insieme intero (3.622 righe).
  Le modifiche ai tre notebook non sono in un commit.

## Cosa è a posto
- Nessuna proteina del test nel training: 0 accession in comune, nessuna osservazione attribuita al test, 813
  proteine aggiunte omologhe al test escluse dall'addestramento.
- Punteggi dei motivi annidati nei fold; standardizzazione sul solo training; early stopping su una partizione
  distinta da quella di valutazione.
- Iperparametri, soglie, α del k-NN ed ensemble scelti solo sull'out-of-fold: il test non entra mai nelle scelte.
- Bootstrap appaiato per le differenze di AUC sul test.
- Calibrazione di Platt in cross-fitting sulle partizioni.
- Varianti predette dal modello del fold in cui la proteina WT era esclusa; CV del supervisionato raggruppata per
  proteina.
- Diversi limiti sono già dichiarati nei notebook (confronto Wefers non appaiato, associazioni di malattia nel verso
  opposto, quanto vale la scheda da sola).

## Non verificato
- Il codice di dettaglio dello 03 per frazioni e percentuali (sezione 5), tabella finale (6), malattie (8) e la
  pagina HTML (~39.000 caratteri di codice): letti solo descrizioni e output.
- Il percorso delle proteine nuove (03, sezione 6b): non è mai stato eseguito (0 proteine in `data/esterni/nuove/`).
- La riproducibilità numerica: non ho rieseguito nulla.

## Ordine consigliato
1. Correggere il collegamento del test HPA (G1) e rieseguire 01 → 02 → 03 da kernel pulito.
2. Caricare il test su DeepLoc 2.0 e inserire i CSV ufficiali (G2).
3. Fissare il modello di sola sequenza come principale; rifare la pulizia della scheda (G4).
4. Riallineare la scelta del modello ai risultati di CV e semplificare (M1).
5. Decidere come presentare varianti (G3) e classi neuronali (M2); togliere la mielina come classe valutata.
6. Verificare o ridimensionare le probabilità dopo la fusione (M3) e la copertura conformale (M4).
