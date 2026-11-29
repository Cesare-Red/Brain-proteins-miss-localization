# NeuroLoc-Δ (try_06) · Verifica della metodologia e del codice dei notebook

Data: 16 novembre 2026. Oggetto: `01_dati.ipynb`, `02_embedding_modello.ipynb`, `03_cervello_varianti_malattie.ipynb`
nello stato attuale della cartella di lavoro (modifiche non ancora in un commit), eseguiti il 15–16 novembre 2026.

**Come è stata fatta.** Lettura integrale del codice dei tre notebook (circa 4.300 righe) e di tutti gli output
salvati; verifiche dirette, con script separati, sui file in `data/` e `cache/`. Il testo dei notebook è stato usato
solo dove descrive il codice attuale: le parti di "diario" sono state ignorate. Nessun notebook è stato rieseguito e
nessun file del progetto è stato modificato (questo report è l'unico file nuovo).

Ogni punto indica da dove viene: **[dati]** = ricalcolato sui file, **[output]** = letto negli output salvati,
**[codice]** = dalla lettura del codice.

`docs/STATO_CODICE.md` e `docs/RISPOSTA_REVISIONE.md` si riferiscono all'esecuzione precedente (test di 292 proteine):
i numeri qui sotto sono quelli dell'esecuzione attuale (test di 741 proteine).

---

## In sintesi

L'impianto di validazione è solido nei punti che contano (nessuna proteina del test nel training, omologhe del test
tolte, standardizzazione e soglie solo sul training, bootstrap appaiato corretto: vedi "Cosa risulta corretto").
I problemi che toccano le affermazioni dell'articolo sono cinque:

1. **Il confronto con DeepLoc 2.0 non esiste nell'esecuzione attuale.** Le predizioni ufficiali non sono state caricate;
   in tabella ci sono solo i valori pubblicati sul test intero, non confrontabili con i nostri.
2. **Il test è il 43% del benchmark originale (741 su 1.717) e non è un campione casuale.** Il collegamento a UniProt ha
   un difetto correggibile, il 38% delle sequenze del test è un frammento della proteina canonica, tre classi su nove
   non sono valutabili.
3. **La classe "Lysosome/Vacuole" ha due definizioni diverse** nelle etichette curate (che includono gli endosomi) e in
   tutte le altre fonti (che li scartano).
4. **Il test non sostiene le affermazioni sull'architettura.** A parità di dati la rete complessa non migliora l'AUC
   rispetto al riferimento semplice (+0,002, IC −0,007…+0,012) e una variante più semplice fa meglio del modello scelto.
   I guadagni misurabili vengono dai dati e dalla scheda testuale.
5. **La tabella finale contraddice una parte delle annotazioni**: il 57% delle proteine lisosomiali curate e l'80–95%
   delle proteine annotate in assone e dendrite ricevono una "probabilità" sotto il 50%.

Seguono problemi di gravità media (varianti, selezione del modello, percentuali, contesto Wefers) e note sul codice.

---

## A · Problemi gravi

### A1 · Confronto con DeepLoc 2.0: assente (02, sezioni 9–10)
- **[output]** Cella della sezione 9: *"Predizioni ufficiali di DeepLoc 2.0: coprono 0% delle 741 proteine del test →
  NON usate"* e *"Confronto con DeepLoc 2.0: NON disponibile"*. **[dati]** In `data/esterni/` non c'è alcun
  `deeploc2_*.csv`.
- Nella tabella `confronto_modelli.xlsx` restano le due colonne "DeepLoc 2.0 ESM1b / ProtT5 pubblicato". Messe accanto
  alle nostre (Jaccard 0,589 contro 0,53; Accuracy 0,453 contro 0,39) sembrano una superiorità, ma non sono
  confrontabili per quattro ragioni: insieme diverso (741 proteine contro 1.717), una sola isoforma per gene (A2),
  metriche su 6 classi con almeno un'etichetta forzata, soglie scelte in modo diverso.
- L'unico riferimento esterno è il contesto di Wefers et al. (B6), non appaiato.
- **Cosa fare.** Caricare `data/hpa_test_per_deeploc2_*.fasta` sul server DTU e rilanciare dalla sezione 9: il codice
  per il confronto appaiato c'è già. Farlo **dopo** la correzione A2, perché cambiando il test cambiano i FASTA. Nella
  tabella dell'articolo togliere i valori pubblicati, o spostarli in una riga di testo con la precisazione.

### A2 · Test HPA: 741 proteine su 1.717, selezionate in modo non casuale (01, cella "Proteine")
Percorso delle proteine del test **[output + dati]**:

| passo | proteine |
|---|---|
| test HPA di DeepLoc 2.0 | 1.717 |
| collegate a UniProt (sequenza identica 319, ENSP 268, 5-meri 806) | 1.393 |
| dopo "una isoforma per gene" | 827 |
| dopo aver tolto i geni già nel training di DeepLoc (41) | 786 |
| nel perimetro cerebrale, > 40 residui | **741** |

- **a) 324 proteine (19%) non collegate; l'avviso del notebook è scattato ed è stato ignorato.** L'output stampa
  *">>> ATTENZIONE: più del 10% del test non è collegato a UniProt"*. La causa non sono "ENSP ritirati", come dice il
  commento. **[dati]** 1.667 ENSP del test su 1.717 sono nella tabella di UniProt, ma solo 584 puntano direttamente a
  una voce Swiss-Prot: gli altri puntano a voci TrEMBL dello stesso gene. Passando per il gene (ENSP → accession →
  ENSG → voce Swiss-Prot con lo stesso ENSG, tutto nello stesso file `uniprot_idmapping_human.dat.gz`) si collegano
  1.666 proteine su 1.717, comprese 312 delle 324 perse.
- La riserva per 5-meri funziona: dove si può confrontare, coincide con la via per gene nel 99,5% dei casi (4 discordi
  su 770). Ma con la via per gene non serve più.
- **Effetto netto, modesto.** I geni distinti passano da 827 a 1.024; 126 dei 201 nuovi sono però già nel training di
  DeepLoc e uscirebbero comunque. Il test crescerebbe di circa 70 proteine (+9%). Il punto è un altro: oggi la
  selezione dipende da un'euristica e l'avviso di controllo è acceso.
- **b) Il benchmark originale contiene più isoforme dello stesso gene** (1.393 proteine collegate → 827 geni; MUC1
  compare 11 volte, sempre con la stessa etichetta). Tenerne una per gene è corretto, ma `drop_duplicates` tiene la
  prima in ordine di file: **[dati]** in 61 geni era disponibile l'isoforma identica alla canonica ed è stata tenuta
  un'altra.
- **c) Molte sequenze del test sono frammenti.** **[dati]** Tra le 741: il 38,5% è più corto della metà della proteina
  canonica, il 18,5% ha meno di 100 residui (mediana 235 residui contro 483 delle proteine di training). L'etichetta
  HPA è del gene (anticorpo), non del frammento: un frammento senza il segnale di smistamento porta un'etichetta che
  nessun modello di sequenza può ricavare. Vale per tutti i modelli, quindi il confronto appaiato resta equo, ma i
  valori assoluti sono depressi. Inoltre struttura AlphaFold e scheda di queste proteine vengono dalla voce canonica,
  mentre la sequenza è il frammento.
- **d) Classi.** Il test misura 6 classi su 9. Lisosoma (1 positivo) e perossisoma (2) non sono valutabili,
  l'extracellulare non è misurato; RE (22) e Golgi (37) hanno pochi positivi e MCC instabili.
- **Cosa fare.** Collegare per gene; a parità di gene scegliere l'isoforma identica alla canonica, altrimenti la più
  lunga; riportare le metriche anche sul solo strato "sequenza identica alla canonica" (oggi 236 proteine); scrivere
  nell'articolo che il test è un sottoinsieme del benchmark e come è stato ottenuto.

### A3 · "Lysosome/Vacuole": le etichette curate includono gli endosomi, le altre fonti no (01, celle UniProt, HPA, GO, LOPIT)
- **[dati]** Delle 468 proteine umane con etichetta DeepLoc "Lysosome/Vacuole", 237 hanno in UniProt solo termini di
  **endosoma** (nessun "lysosome" o "vacuole"). Quindi la classe curata comprende gli endosomi (da confermare sul
  materiale supplementare di DeepLoc).
- **[codice]** Lo 01 fa il contrario in tutte le altre fonti: le regole UniProt e GO mettono `endosom` e `vesicle`
  tra i termini "non organello"; HPA mappa solo "Lysosomes" (non "Vesicles" né "Endosomes"); OpenCell scarta
  "vesicles"; i marcatori LOPIT degli endosomi finiscono in "altro".
- **Conseguenze misurate [dati]:**

  | fonte, classe lisosoma | sensibilità contro DeepLoc | (per confronto, altre classi) |
  |---|---|---|
  | UniProt forte | 0,54 | 0,96–1,00 |
  | GO sperimentale | 0,28 | 0,54–0,94 |
  | LOPIT-DC | 0,19 | 0,15–0,83 |
  | HPA Enhanced/Supported | 0,02 | 0,36–0,87 |

  Con la regola delle assenze, una proteina endosomiale aggiunta (178 nel pool esteso) riceve "assente dal lisosoma" da
  UniProt, mentre la definizione curata direbbe "presente": la rete riceve segnali opposti per la stessa classe.
  È una spiegazione concreta del perché il lisosoma è l'organello peggiore fuori fold (AUPRC 0,36) e della tabella
  finale (A5).
- Non tocca il test HPA (il lisosoma lì non è valutato), tocca l'AUPRC macro in CV (criterio di scelta) e il prodotto
  finale.
- **Cosa fare.** Allineare la definizione: mappare gli endosomi su Lysosome/Vacuole in tutte le fonti (schema DeepLoc),
  oppure dichiarare la classe "lisosoma in senso stretto" e ricostruire di conseguenza anche l'etichetta curata.

### A4 · Il test non sostiene le affermazioni sull'architettura (02, sezioni 7–10)
Valori dall'output della sezione 10 **[output]** (test: 741 proteine, AUC e AUPRC macro su 6 classi):

| modello | AUPRC CV | AUC test | Δ AUC contro "stessi dati" (IC 95%) | AUPRC test |
|---|---|---|---|---|
| Riferimento semplice, solo DeepLoc | 0,592 | 0,816 | −0,021 | 0,519 |
| Riferimento semplice, **stessi dati** | 0,606 | 0,837 | — | 0,538 |
| Rete, 1 seme | 0,648 | 0,838 | +0,001 (−0,010 … +0,011) | 0,573 |
| Variante: solo medie + descrittori + struttura | 0,639 | **0,846** | +0,009 (+0,001 … +0,017) | 0,572 |
| NeuroLoc-LA (attenzione sui residui) | 0,631 | 0,832 | −0,005 (−0,016 … +0,006) | 0,565 |
| **Scelto: ensemble rete + NeuroLoc-LA** | 0,662 | 0,840 | +0,002 (−0,007 … +0,012) | 0,576 |
| Rete + scheda testuale (non solo sequenza) | 0,683 | 0,859 | +0,022 (+0,011 … +0,032) | 0,612 |

- **Sull'AUC, a parità di dati, il modello scelto non è distinguibile dal riferimento semplice** (media di ProtT5 +
  percettrone). Il notebook lo stampa, ma l'articolo non può presentare l'architettura come contributo.
- **I pezzi complessi non aggiungono nulla di misurabile**: la variante con le sole medie fa meglio del modello scelto
  sull'AUC e uguale sull'AUPRC (0,572 contro 0,576); senza struttura AlphaFold la CV è più alta (0,652 contro 0,648);
  NeuroLoc-LA da sola è peggiore della rete, e nell'ensemble porta +0,004 in CV e +0,0001 sul test.
- **La classifica in CV non predice quella sul test.** In CV la rete stacca il riferimento "stessi dati" di +0,04–0,06
  di AUPRC, sul test di +0,002 di AUC. La CV misura 9 classi con etichette di origine UniProt, il test 6 classi con
  etichette HPA: il criterio di scelta non è allineato al banco di prova.
- **Sull'AUPRC del test una differenza c'è** (0,573–0,576 contro 0,538), ma non ha intervallo: il bootstrap appaiato è
  calcolato solo sull'AUC, mentre la scelta si fa sull'AUPRC. Sulle metriche a soglia il quadro è misto (Jaccard:
  riferimento 0,597, scelto 0,589; Macro F1: 0,495–0,503 contro 0,523).
- **Dimensione dell'ensemble.** Il modello scelto media 25 reti (3 semi × 5 fold + 2 × 5), i riferimenti 5: una parte
  del piccolo vantaggio è solo media di più modelli.
- **Regola di scelta.** Per la regola del notebook stesso (entro 2 deviazioni standard = equivalenti) "Rete neurale" è
  equivalente all'ensemble: a parità, andrebbe scelto il più semplice.
- **Cosa si può affermare con questi numeri:** più dati aiutano (+0,021 sul riferimento, +0,022 sulla rete); le fonti
  HPA nel training valgono +0,008 (in parte allineamento alla fonte del test, già dichiarato); la scheda testuale vale
  +0,021–0,022 ma non è "sola sequenza".
- **Cosa fare.** Bootstrap appaiato anche sull'AUPRC macro; ablazioni con almeno 3 semi (oggi una sola esecuzione
  ciascuna, contro un vincitore scelto come massimo di cinque); riportare in CV anche l'AUPRC sulle sole 6 classi del
  test; presentare come modello principale il più semplice tra gli equivalenti.

### A5 · La tabella finale contraddice una parte delle annotazioni (03, sezioni 4 e 6)
`probabilita_proteine_cervello.xlsx` è il prodotto per i ricercatori. **[dati]** Confronto con le etichette note:

| classe | positivi curati (DeepLoc) | con "% probabilità" < 50% | < 10% |
|---|---|---|---|
| Lysosome/Vacuole | 468 | 57,5% | 29,1% |
| Endoplasmic reticulum | 637 | 29,8% | 7,7% |
| Golgi apparatus | 419 | 24,8% | 4,1% |
| Cell membrane | 1.227 | 18,3% | 5,1% |
| Mitochondrion | 644 | 16,9% | 8,5% |
| Nucleus / Cytoplasm | 2.458 / 2.635 | 7,0% / 7,7% | < 1% |

- **Causa [codice].** L'etichetta DeepLoc è esclusa dalla fusione (`FUSE_ALL`): per le proteine curate la tabella usa
  UniProt forte e le altre misure, pesate con coefficienti stimati sulle sole fonti indipendenti. Esempio dagli output:
  **SOD1, mitocondrio 4%**, con DeepLoc, MitoCarta e GO sperimentale che dicono mitocondrio. Per il lisosoma pesa anche A3.
- La verifica dopo la fusione (ECE 0,008, "errori sicuri" 1,15%) è corretta ma è una media su tutte le coppie
  proteina-classe, quasi tutte negative: non mostra questo effetto sui positivi.
- **Compartimenti neuronali [dati]:**

  | classe | annotate da almeno una fonte | di cui con P < 50% | proteine con P ≥ 50% | di cui già annotate |
  |---|---|---|---|---|
  | Assone | 289 | 231 (80%) | 64 | 58 |
  | Dendrite | 244 | 231 (95%) | 13 | 13 |
  | Presinapsi | 754 | 238 (32%) | 565 | 516 |
  | Postsinapsi | 983 | 206 (21%) | 845 | 777 |
  | Mielina | 7 | 6 | 1 | 1 |

  Per assone e dendrite la tabella nega quasi tutte le annotazioni esistenti; per le sinapsi le predizioni sopra il 50%
  sono per il 91–92% proteine già in SynGO (nuove: 49 e 68). Casi di studio dagli output: **tau (MAPT) assone 34%,
  dendrite 92%**; PLP1 mielina 2%, MOG 0%, MBP 28%.
- **Causa [codice].** Per le classi neuronali il peso delle misure è la media di quelli degli organelli (0,49) e le
  sensibilità sono quelle del canale di rumore, che lo 02 stesso dichiara non identificabili; con la regola delle
  assenze una fonte che annota la proteina altrove conta come prova contraria. Il notebook dichiara queste probabilità
  "non verificate": la verifica qui sopra dice che per assone e dendrite non sono utilizzabili così.
- **Cosa fare.** Decidere cosa mostra la tabella per le proteine già annotate (l'annotazione accanto alla predizione di
  sola sequenza fuori fold, non una fusione che la contraddice). Per le classi neuronali mostrare l'ordinamento di sola
  sequenza e l'annotazione, senza fusione, oppure togliere le assenze dalla fusione. I compartimenti neuronali sono
  presentati come valore aggiunto del lavoro: oggi quel valore non è dimostrato.

---

## B · Problemi di gravità media

### B1 · Varianti (03, sezione 7)
- **Il testo e l'output non coincidono.** Il markdown dice che gli intervalli dei segnali di spostamento "contengono
  0,5"; l'output elenca due segnali su otto che lo escludono. **[dati]** Entrambi sono artefatti:
  `rischio_ritenzione_RE` contiene l'LLR per costruzione (correlazione di Spearman 0,44 con −LLR sulle varianti
  misurate); `|ddG_inserzione_TM|` è diverso da zero in 7 varianti su 242. Il risultato negativo regge, ma il verdetto
  automatico va corretto: un segnale che contiene l'LLR non è "di spostamento", e servono un numero minimo di varianti
  informative e una correzione per i 13 segnali provati.
- **Su ClinVar il punteggio supervisionato è l'LLR.** **[dati]** Il 76,5% delle varianti P/LP o B/LB sta in proteine
  con una sola classe: l'AUROC complessiva mescola gene e variante. `score_variante` supera l'LLR di +0,022
  (IC 0,013–0,029) sull'insieme, ma **entro la stessa proteina** vale 0,886 contro 0,886 **[output]**. Inoltre
  `score_variante` non contiene "solo ciò che cambia con la mutazione": i rischi per meccanismo moltiplicano per le
  probabilità della proteina normale.
- **Varianti misurate:** 242 varianti in 82 proteine, solo 45 con entrambi gli esiti. Le 113 non classificate P/LP o
  B/LB ricevono il punteggio da un modello addestrato su tutte le varianti ClinVar (anche della stessa proteina), le
  altre fuori fold: protocollo misto.
- **[dati]** Il bootstrap è sulle varianti, non sulle proteine. Ricalcolato per proteina l'intervallo dell'LLR sulle
  misurate è 0,534–0,684 (contro 0,535–0,677): la conclusione non cambia, ma il bootstrap per proteina è quello giusto.
- **[codice]** L'LLR è quello "wild-type marginal" (sequenza non mascherata), calcolato in fp16: va dichiarato.

### B2 · Stime in CV ottimistiche e ablazioni dentro il rumore (02, sezioni 7–8)
- **[codice]** 48 configurazioni provate sul fold 0, le 5 migliori in CV, poi α del k-NN, soglie, ensemble e modello
  finale scelti sulle stesse predizioni fuori fold. Il test non entra mai (corretto), ma l'AUPRC in CV del modello
  scelto è ottimistica e le differenze con le ablazioni favoriscono il vincitore.
- **[output]** Deviazione standard tra semi 0,0023. Tre ablazioni su cinque stanno entro 0,005 dal vincitore (senza
  struttura è sopra). Le altre due superano il rumore in CV (solo medie −0,008, solo ESM-2 −0,012), ma sul test "solo
  medie" è la migliore: vedi A4.
- **Righe duplicate [output]:** "Rete, pool esteso" è identica a "Rete, 1 seme" (stessa configurazione e stesso seme:
  lo stadio `cervello` riaddestra lo stesso modello); "Rete neurale + k-NN" è identica a "Rete neurale" (α = 0). Vanno
  tolte da tabelle e figure.

### B3 · Controllo dell'omologia: buono, ma tra candidati (02, sezione 4)
- **[codice]** Si allineano solo i 3 candidati per 5-meri e i 5 per coseno; identità locale ≥ 30% su almeno metà della
  proteina più corta. Non è il criterio di DeepLoc (identità globale, tutti contro tutti): dichiarato.
- **[dati]** Controllo indipendente: nessuna proteina del test ha più dell'80% dei suoi 5-meri in una proteina di
  training usata (una sola supera il 50%). Il 20,8% del test ha un'omologa curata, con identità mediana 0,37 (quattro
  sopra 0,6): lo strato è riportato (AUC 0,855 contro 0,837).
- Una conferma con MMseqs2 tutti contro tutti chiuderebbe il punto davanti a un revisore.

### B4 · Valutazione "nel cervello" e calibrazione (03, sezioni 2–4)
- **[codice]** Riferimento e calibrazione usano le stesse fonti dell'addestramento, con la regola delle assenze: le
  probabilità sono tarate sulla frequenza di *annotazione*, non di presenza. Per le classi neuronali è un problema di
  positivi e non etichettati (dichiarato): l'ECE dopo la scala di Platt (0,006–0,008) misura l'accordo con le
  annotazioni, non la verità.
- **[output]** Prima della calibrazione la rete sovrastima molto le sinapsi (ECE 0,29 e 0,38): è l'effetto atteso
  della specificità fissata a 1 nel canale.

### B5 · "Ripartizione %" (03, sezioni 5–6)
- **[codice]** Sono misurate e valutate solo le frazioni citosolica e nucleare (Spearman fuori fold 0,765 e 0,644,
  tetto tra esperimenti 0,881 e 0,801). Il resto è ripartito tra gli organelli in proporzione alle probabilità di
  presenza: è l'operazione che il notebook stesso dichiara scorretta ("una probabilità di presenza non è una
  ripartizione"), e non ha alcuna verifica.
- Le misure vengono da HEK293T e HeLa, non da neuroni. Nell'articolo le percentuali degli organelli vanno presentate
  come euristica, o tolte.

### B6 · Contesto Wefers / test HOU (03, sezione 12)
- **[output]** 3.485 proteine HOU su 3.622 sono nel nostro training: si valutano fuori fold, ma le loro etichette
  vengono dalle stesse banche dati del nostro addestramento e la scelta del modello è stata fatta sulle predizioni
  fuori fold di queste stesse proteine.
- **[codice]** Il "cytosol" di HOU è confrontato con il nostro "Cytoplasm", che include il citoscheletro (in HOU è una
  classe a parte). "Dentro l'intervallo" usa solo la nostra incertezza, non la loro.
- È dichiarato "non appaiato": va tenuto come contesto, fuori dal sommario e dalle conclusioni.

### B7 · Altri punti
- **Malattie (03, sezione 8).** È un controllo di buon senso: 5 attese ritrovate su 10, 2 nel verso opposto. Il gruppo
  "encefalopatie energetiche" (AUC 0,97) è definito da nomi di malattia mitocondriale: ripete la bontà della classe
  mitocondrio, non aggiunge un'evidenza.
- **Perimetro (01).** Passa l'80% di Swiss-Prot. **[codice]** `neuron_ncpm >= BRAIN_NTPM` applica la soglia in nTPM a
  una misura in nCPM. Le annotazioni neuronali delle proteine sotto 5 nCPM nei neuroni vengono cancellate.
- **Scheda testuale.** Il simbolo del gene resta nella scheda (MRPL…, TOMM…, PEX…, NDUF…, SLC…): è un indizio di
  localizzazione. Conta solo per la variante `con_scheda`, da descrivere come "conoscenza dei curatori".
- **Pesi dei positivi.** **[codice]** Si applicano solo alla fonte curata: le classi neuronali, le più rare, non hanno
  alcun bilanciamento, al contrario di quanto dice l'intestazione dello 01.

---

## C · Codice e riproducibilità

- **Esecuzione non pulita.** Contatori: 01 da 15 a 28; 02 da 1 a 8, poi 15–25 e 27–34 (celle rilanciate; la cella
  AlphaFold è durata 3 h 50 min); 03 pulito (1–15). Serve un "Restart & Run All" dello 01 e dello 02, e il commit.
- **README non aggiornato:** dice che i notebook sono senza output e descrive lo stato precedente.
- **Markdown non aggiornato nel testo corrente** (non nel diario): lo 03 cita `abl_senza_card` (non esiste più) e un
  richiamo "0,81–0,85" (ora 0,83–0,87); i commenti delle celle ESM-2 e ProtT5 parlano di GTX 1080 e di uguaglianza
  "bit per bit", ma l'esecuzione è in fp16 su A5000.
- **Preprocessing sul test (non supervisionato).** La PCA degli alberi e quella dei residui sono stimate su tutte le
  proteine, test compreso. Effetto trascurabile, ma va stimata sul training o dichiarata. L'early stopping degli
  alberi usa una divisione casuale, non le partizioni di omologia.
- **ProtT5:** un batch fallito viene saltato con un avviso e le sue righe restano a zero. In questa esecuzione
  "completo: True"; conviene un `assert` su `DONE` e `DONE5` prima di procedere.
- **ClinVar (01):** se la stessa sostituzione ha classi discordi (cambi nucleotidici diversi) resta la prima in ordine
  di file; la quota leave-one-out usa conteggi presi prima della deduplicazione. Effetto piccolo.
- **Proteine nuove (03, sezione 6b):** mai eseguita (0 proteine). Usa una lista di pulizia diversa dallo 01 (con il
  difetto `import\w*` che cancella "important") e include le Keywords: la scheda di una proteina nuova non è costruita
  come quelle di training.
- **Versioni dei dati:** `versioni_dati.csv` registra data del file e SHA-256; la versione di HPA non è registrata
  (il test di DeepLoc è HPA del 2022, le etichette di training sono HPA di novembre 2026). `requirements.lock.txt` manca.

---

## Cosa risulta corretto (verificato)

- **Nessuna proteina del test nel training [dati]:** 0 accession in comune, 0 sequenze identiche, nessun quasi-duplicato
  per 5-meri; le osservazioni delle proteine del test non entrano mai tra le etichette.
- **Omologhe del test fuori dal training [output]:** 1.074 proteine aggiunte tolte, più 938 con un'omologa curata in un
  altro fold (escluse anche dalle valutazioni fuori fold).
- **Niente informazione del fold di valutazione nell'addestramento [codice]:** standardizzazione sul solo training,
  early stopping su una partizione distinta, punteggi di motivo annidati, k-NN con soli riferimenti fuori fold, scelta
  dell'ensemble annidata.
- **Bootstrap appaiato [codice]:** l'AUC pesata con i pari merito è implementata correttamente; la correzione di
  Benjamini-Hochberg nelle malattie anche.
- **Varianti [codice]:** proteina normale e mutante predette dal modello del fold in cui la proteina era esclusa; CV
  raggruppata per proteina; posizioni nella sequenza troncata e finestre ±15 coerenti tra 01, 02 e 03; la quota di
  patogeniche del gene non entra tra le feature.
- **Catena dei file [dati]:** date coerenti 01 → 02 → 03; lo stadio `cervello` riproduce esattamente il vincitore della
  CV (0,6477), quindi l'addestramento è deterministico su questa macchina.
- **Onestà degli output:** il notebook stampa già diversi risultati sfavorevoli (confronto con DeepLoc non disponibile,
  guadagno nullo a parità di dati, attese sulle malattie non ritrovate, risultato negativo sulle varianti).

## Non verificato

- Nessuna riesecuzione: la riproducibilità numerica non è stata provata.
- Il codice JavaScript della pagina HTML (circa 370 righe) e la sezione 6b dello 03.
- I valori pubblicati di DeepLoc 2.0 e di Wefers et al. riportati nelle tabelle non sono stati ricontrollati sugli
  articoli.
- L'hook sul layer intermedio di ESM-2 e ProtT5 (02) dipende dal formato di uscita dei layer nella versione di
  `transformers` installata: non provato.
- L'indipendenza dei marcatori LOPIT dalle altre fonti.

---

## Ordine consigliato

1. **01:** collegare il test per gene, scegliere l'isoforma canonica, sistemare la soglia nCPM (A2, B7).
2. **01:** decidere la definizione di Lysosome/Vacuole e allineare le fonti (A3).
3. Rieseguire 01 → 02 da kernel pulito; caricare i nuovi FASTA su DeepLoc 2.0 e rilanciare la sezione 9 (A1).
4. **02:** bootstrap anche sull'AUPRC, ablazioni con più semi, togliere le righe duplicate, scegliere il modello più
   semplice tra gli equivalenti (A4, B2).
5. **03:** rivedere cosa mostra la tabella per le proteine annotate e per le classi neuronali (A5); correggere il
   verdetto delle varianti (B1); dichiarare euristiche le percentuali (B5).
6. Solo dopo, riallineare la bozza.

## Nota sulla bozza (`docs/NeuroLoc_articolo_revisione.docx`)

Non è stata confrontata punto per punto. Dalla lettura dell'inizio, tre cose da cambiare oltre ai numeri (la bozza ha
ancora 287 proteine di test, 6.275 curate e la Tabella 1 dell'esecuzione precedente):
- "profilazione testuale da cui è stata tolta ogni informazione di localizzazione": lo 02 misura il contrario (la sola
  scheda vale 0,553 di AUPRC contro 0,593 dei soli embedding di sequenza);
- la rappresentazione "sequenza, singoli residui e testo" non è il modello principale, che ora è di sola sequenza, e i
  residui non portano guadagno (A4);
- la Sezione 4 "confronto con lo stato dell'arte" oggi non ha un confronto da riportare (A1).
