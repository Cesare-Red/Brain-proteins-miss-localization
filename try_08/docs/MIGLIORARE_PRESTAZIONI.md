# NeuroLoc-Δ · Come migliorare le prestazioni rispetto ai metodi pubblicati

Analisi del 19 novembre 2026. Le leve sono ordinate per **guadagno atteso diviso costo**. Per ognuna: perché dovrebbe
funzionare, cosa toccare e cosa costa. I numeri di guadagno sono stime dalla letteratura o da prove precedenti di questo
progetto: vanno verificati con la CV e con il test.

## Dove si gioca il confronto
- **Test HPA (DeepLoc 2.0, Tabella 2).** Le metriche pubblicate sono quasi tutte **a soglia**: accuratezza, Jaccard,
  F1, MCC. Contano quindi il ranking (AUC) e anche la scelta delle soglie e del numero di etichette per proteina.
- **I metodi pubblicati sono addestrati su più dati.** DeepLoc 2.0 usa circa 28.000 proteine eucariote; noi circa 6.400
  curate umane più le aggiunte. A parità di modello, i dati sono la leva più grande.
- **Molte sequenze del test HPA sono frammenti** (isoforme Ensembl, colonna `test_copertura`). Un frammento senza la
  sua estremità N-terminale perde il segnale di smistamento.

## 1. Addestrare anche sulle altre specie di DeepLoc (costo minimo, guadagno alto)
`ALTRE_SPECIE = False` nello 01 esclude circa 22.000 proteine eucariote con etichette curate. Una prova precedente di
questo progetto (intestazione dello 01) misurava **circa +0,05 di AUPRC** sul test umano tenendole con peso ridotto. La
griglia dello 02 ha già il parametro `peso_non_umani` (0,3).

Il perimetro resta umano per le **predizioni** e le **valutazioni**: le altre specie servono solo ad addestrare. I
segnali di smistamento degli organelli sono conservati negli eucarioti. Le partizioni di DeepLoc coprono anche le altre
specie, e il test HPA è già senza omologhe oltre il 30% verso tutto il set di DeepLoc.

**Cosa toccare:**
- nello 01, `ALTRE_SPECIE = True`;
- nello 02 il peso dei non umani nella perdita c'è già (`peso_non_umani`, `HUMAN_t`), ma vanno sistemati due punti:
  - le metriche di CV e la scelta del modello devono restare sulle sole curate umane (`human_gold`): oggi diverse celle
    usano `gold_rows`, che includerebbe le altre specie;
  - il perimetro "cervello" non si applica alle altre specie: vanno escluse dalle tabelle dello 03.

Il tempo degli embedding sale di circa 2,5 volte.

## 2. Soglie e numero di etichette pensati per le metriche pubblicate (costo minimo)
Oggi la soglia per classe massimizza l'MCC, come in DeepLoc. Il Jaccard e l'accuratezza premiano anche il **numero
giusto di etichette** per proteina (vere: 1,22). Due passi, entrambi stimati sulle predizioni fuori fold delle curate,
mai sul test:
- soglie per classe scelte per il Jaccard medio (ricerca a coordinate partendo dalle soglie MCC);
- una regola sul numero di etichette: la classe migliore sempre, la seconda solo se supera la sua soglia con un margine
  scelto in CV.

È la leva più diretta sulle colonne Accuracy e Jaccard della Tabella 2.

## 3. Attenzione sui residui di ProtT5, con i dati in più e il canale di rumore (costo medio)
Gli embedding per residuo di ProtT5 ora esistono già (`data/res_prott5_4k.npy`, sezione 8d). La nostra rete migliore
lavora su riassunti fissi; DeepLoc 2.0 e LAProtT5 imparano **dove guardare**.

Un'attenzione leggera su quei residui, addestrata sul pool esteso con il canale di rumore, unisce i due vantaggi: i
nostri dati e la loro aggregazione. In più si può aggiungere la supervisione dei segnali di smistamento (la KL di
DeepLoc, file già scaricato), che concentra l'attenzione sui segnali veri. NeuroLoc-LA lo fa già sugli embedding di
ESM-2 ridotti con PCA; su ProtT5 intero, la variante a parità di condizioni dice quanto vale.

## 4. Ritagli casuali per i frammenti (costo basso-medio)
Durante l'addestramento si mostrano anche **pezzi** di sequenza (inizio, fine, finestre interne), come fa Wefers con
le due metà. La rete impara a usare indizi che restano nei frammenti: domini, composizione, eliche TM.

Lo strato "frammento (meno di metà)" del test (02, sezione 10) misura direttamente il guadagno.

## 5. Più modelli di linguaggio nell'ensemble (costo medio)
ESM-2 650M e ProtT5 sono già dentro. I candidati, pubblici e scaricabili come gli altri:
- **ESM-2 3B**;
- **ESM C 600M** (EvolutionaryScale, 2024);
- **ProstT5**, che aggiunge la struttura come "lingua" (token 3Di).

Rappresentazioni diverse sbagliano in modo diverso, e l'ensemble scelto in CV ne approfitta. Ogni modello in più costa
un passaggio di embedding.

## 6. Affinare il modello di linguaggio con LoRA (costo alto, guadagno potenzialmente alto)
Gli embedding oggi sono congelati. L'affinamento con LoRA di ProtT5 o ESM-2 sui compiti di localizzazione migliora le
prestazioni rispetto agli embedding fissi (Schmirler et al., *Nat Commun* 2024: "Fine-tuning protein language models
boosts predictions across diverse tasks").

Su una RTX A5000: ESM-2 650M con LoRA è fattibile, ProtT5-XL in mezza precisione è al limite. Ore di calcolo per ogni
fold, e serve la stessa disciplina di partizioni e di scelta annidata.

## 7. Il training set combinato HPA + UniProt di Wefers et al. (costo medio, dati)
Su Zenodo (record 18435290) ci sono i loro training set umani, `hpa_uniprot_combined_human_trainset.csv`, costruiti con
etichette concordanti tra banche dati. Entrano come una fonte in più nello 01. Il canale di rumore ne stima
l'affidabilità.

Attenzione al test HOU: quel training è costruito per essere non omologo al test HOU, ma le nostre partizioni vanno
ricontrollate.

## Cosa NON fare
- Scegliere modello o soglie guardando il test (HPA o HOU): i numeri non sarebbero più confrontabili con quelli
  pubblicati.
- Aggiungere la scheda testuale al modello da confrontare: è conoscenza dei curatori, non sequenza. I metodi pubblicati
  non la usano.
- Pesare di più la fonte HPA per "allinearsi" al test HPA, senza dichiararlo.

## Ordine consigliato
1. Leve 1 e 2: un'esecuzione, nessun codice nuovo di rilievo.
2. Leve 3 e 4: riusano gli embedding per residuo già calcolati.
3. Leve 5 e 7: più dati e più rappresentazioni.
4. Leva 6: solo se le precedenti non bastano.
