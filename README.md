# 🧠 NeuroLoc-Δ
### Predizione della localizzazione subcellulare di proteine cerebrali umane

[![Python 3.11](https://img.shields.io/badge/Python-3.11-blue?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.x-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![HuggingFace](https://img.shields.io/badge/HuggingFace-Transformers-FFD21E?logo=huggingface&logoColor=black)](https://huggingface.co/)
[![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)](https://jupyter.org/)
[![License: MIT](https://img.shields.io/badge/Licenza-MIT-green)](LICENSE)

---

## 🎯 Obiettivo

NeuroLoc-Δ è un sistema computazionale per la predizione della **localizzazione subcellulare di proteine cerebrali umane**, con estensione ai compartimenti anatomici specifici del neurone.

### Il problema che risolve

I tool esistenti per la predizione della localizzazione subcellulare — tra cui lo stato dell'arte **DeepLoc 2.0** — sono strumenti generalisti: operano su proteine di tutti gli eucarioti e restituiscono una classificazione in 9 compartimenti standard (citoplasma, nucleo, mitocondrio, ecc.). Questo approccio è potente e ben validato, ma presenta un limite strutturale per le neuroscienze:

> **Il neurone è la cellula morfologicamente più complessa dell'organismo.** Possiede strutture — assone, dendriti, terminale presinaptico, densità postsinaptica, guaina mielinica — che non esistono in nessun altro tipo cellulare e che sono irrilevanti per un classificatore generico.

Sapere che una proteina "è nel citoplasma" non dice nulla al neurologo che vuole capire se quella proteina è all'assone o al corpo cellulare, se è alla presinapsi o alla postsinapsi. Questa distinzione è fondamentale per comprendere le sinaptopatie, le malattie neurodegenerative e l'interpretazione delle varianti genetiche nei pazienti.

NeuroLoc-Δ estende la classificazione standard a **14 compartimenti**, aggiungendo 5 classi neuronali specifiche. È addestrato esclusivamente su proteine umane espresse nel cervello, escludendo contaminanti plasmatici e ottimizzando lo spazio di rappresentazione per il dominio neurologico.

Il sistema include inoltre il modulo **NeuroLoc-Δ** propriamente detto: data una variante missense (ad es. da ClinVar), predice se e come quella sostituzione amminoacidica altera il profilo di localizzazione della proteina, identificando il meccanismo molecolare rotto (perdita del ligando PDZ, destabilizzazione di un'elica transmembrana, alterazione della presequenza mitocondriale, ecc.).

---

## 🔬 Metodologia

### Il sistema a 14 classi

Le classi predette sono organizzate in due livelli:

| Livello | Classi | Applicabilità |
|---|---|---|
| **Generali** (9) | Citoplasma, Nucleo, Extracellulare, Membrana plasmatica, Mitocondrio, Reticolo Endoplasmatico, Lisosoma/Vacuolo, Golgi, Perossisoma | Tutte le proteine cerebrali |
| **Neuronali** (5) | Assone, Dendrite, Presinapsi, Postsinapsi, Mielina | Solo proteine espresse in neuroni (Assone–Postsinapsi) o oligodendrociti (Mielina) |

La classificazione è **multi-label**: una proteina può essere localizzata in più compartimenti simultaneamente (es. SHANK3 è sia postsinaptica che dendritica). L'output non è una classe singola, ma un vettore di probabilità calibrate per ciascuno dei 14 compartimenti.

---

### Il doppio encoder di sequenza: ESM-2 + ProtT5

Il sistema utilizza due protein language model (PLM) distinti in parallelo, ciascuno con architettura e corpus di addestramento differenti:

**ESM-2 650M** (`facebook/esm2_t33_650M_UR50D`)
Modello di tipo BERT (encoder-only) addestrato con masked language modeling su 250 milioni di sequenze UniRef50. Eccelle nel catturare le relazioni locali tra residui vicini nella sequenza, ed è particolarmente sensibile ai segnali di sorting corti (peptide segnale, motivi terminali come PTS1, KDEL, ligandi PDZ).

**ProtT5-XL** (`Rostlab/prot_t5_xl_half_uniref50-enc`)
Modello di tipo T5 (encoder-decoder, solo encoder usato) addestrato su 2.1 miliardi di sequenze BFD + UniRef50. Cattura proprietà globali della proteina — tendenza al disordine, famiglia strutturale, caratteristiche dell'intera catena — complementari a quelle di ESM-2.

La complementarità non è un'ipotesi: nel paper di DeepLoc 2.0, passare da ESM-1b a ProtT5 vale +5 punti di Jaccard. ESM-2 è architetturalmente superiore a ESM-1b. Usare entrambi in fusione permette di sfruttare i punti di forza di ciascuno senza perdere informazione.

Per ogni sequenza, ciascun encoder produce **8 riassunti** (pooling biologicamente motivato):

| Riassunto | Residui | Segnale biologico catturato |
|---|---|---|
| `mean` | Intera sequenza | Rappresentazione globale |
| `mid` | Layer intermedio (hook) | Caratteristiche intermedie del fold |
| `max` | Intera sequenza | Picchi di attivazione |
| `n20` | Primi 20 aa | Peptide segnale, miristoilazione N-term |
| `n60` | Primi 60 aa | Presequenza mitocondriale |
| `centro` | Terzo centrale | Domini globulari, NLS interni, eliche TM |
| `c15` | Ultimi 15 aa | PTS1, KDEL, KKXX, ligandi PDZ |
| `c50` | Ultimi 50 aa | Code citoplasmatiche, YxxΦ, dileucina |

Il pooling su finestre biologicamente definite è motivato dal fatto che i segnali di sorting sono brevi (3–60 aa) e posizionalmente specifici: una media globale li diluisce irrimediabilmente.

---

### Il modulo testuale: PubMedBERT su `card_protein`

Per ogni proteina viene costruita una **scheda testuale funzionale** (`card_protein`) che raccoglie — in inglese, la lingua dei modelli biomedici — nome, famiglia, funzione, attività catalitica, vie metaboliche, domini, partner, modifiche post-traduzionali, processi biologici GO, funzioni molecolari GO, espressione cerebrale, malattie associate.

La scheda viene codificata da **PubMedBERT** (`NeuML/pubmedbert-base-embeddings`), un modello addestrato su coppie di frasi di PubMed, producendo un vettore da 768 dimensioni. Le schede più lunghe di 512 token vengono lette a blocchi sovrapposti (overlap 64 token) e mediate.

**Precauzione critica contro il data leakage:** dalla scheda vengono rimosse sistematicamente tutte le parole che nominano compartimenti o processi di localizzazione (≈80 pattern: `nuclear`, `mitochondrial`, `axon*`, `synap*`, `signal peptide`, `transmembrane`, ecc.). Senza questa pulizia, il modello leggerebbe direttamente la risposta: test interni mostrano che con i nomi non ripuliti il guadagno apparente raddoppia, ma metà è spiegato dalla parola "Mitochondrial" nel nome della proteina.

L'aggiunta del testo ripulito vale circa **+4 punti di macro-AUPRC** rispetto alla sola sequenza.

---

### I descrittori strutturali: AlphaFold

Per ogni proteina viene scaricato il modello strutturale da AlphaFold DB (formato mmCIF, v4) e calcolati **15 descrittori geometrici** dai soli atomi Cα e dal pLDDT (la stima di confidenza di AlphaFold, usata come proxy del disordine intrinseco):

- **Confidenza:** pLDDT medio, deviazione standard, frazione < 50 (disordine), frazione ≥ 90 (ben ripiegata), pLDDT dei 30 residui N-terminali e C-terminali
- **Compattezza:** raggio di girazione normalizzato per la lunghezza (globulare vs esteso)
- **Forma:** anisotropia e asfericitià del tensore di girazione (formula canonica Theodorou-Suter 1985)
- **Complessità:** densità di contatti Cα-Cα (< 8 Å, sep ≥ 3) e ordine di contatto relativo
- **Disordine:** frazione del tratto disordinato più lungo

Il pLDDT basso è il miglior predittore computazionale disponibile di disordine intrinseco: le proteine molto disordinate tendono al nucleo, ai condensati e ai granuli di RNA. Questa informazione non è contenuta nella sequenza lineare in modo facilmente estraibile dai PLM.

Dove AlphaFold DB non è raggiungibile o la proteina non ha un modello, i descrittori vengono azzerati e un flag `struttura_presente = 0` segnala la modalità mancante alla rete.

---

### La fusione multimodale

I quattro rami (ESM-2 × 8, ProtT5 × 8, PubMedBERT × 1, AlphaFold × 1, più descrittori manuali e punteggi di motivo terminale) vengono fusi con un'architettura **a token**:

1. Ogni blocco di input passa per una proiezione lineare indipendente + LayerNorm → diventa un token di dimensione T
2. Con `fusione = 2`: due strati di Transformer (4 teste di attenzione) fanno interagire i token prima della classificazione
3. Con `fusione = 0`: concatenazione diretta (MLP classico, ma con ogni sorgente già normalizzata alla stessa scala)

La fusione a token risolve il problema della disparità dimensionale: senza proiezione, i ~10.000 numeri degli embedding di sequenza schiaccierebbero i 768 della scheda testuale. Il **dropout di modalità** (block dropout) spegne interi token durante il training, insegnando alla rete a funzionare anche con schede povere o strutture mancanti.

L'uscita del classificatore è un vettore multi-task:
- **14 probabilità di localizzazione** (9 generali + 5 neuronali)
- **18 target ausiliari** (segnali di sorting: peptide segnale, transit peptide, eliche TM, ancore lipidiche, NLS, glicosilazione, superclassi gerarchiche)
- **2 frazioni continue** (citosolica e nucleare, training su misure di frazionamento da Hein 2024 e Itzhak 2016)

---

### Il canale di rumore multi-fonte

Il training integra 11 fonti eterogenee (DeepLoc, UniProt forte/debole, HPA, OpenCell, MitoCarta, LOPIT × 3, BioID, SynGO) attraverso un **canale di rumore** che apprende per ogni fonte e classe la sensibilità (quante localizzazioni vere riporta) e la specificità (quanti falsi positivi). Questo permette di usare anche etichette deboli o incomplete senza esserne ingannati: le fonti meno affidabili contribuiscono di meno alla loss, automaticamente.

---

## 📊 Confronto con lo stato dell'arte

### Tabella comparativa

| Caratteristica | **DeepLoc 2.0** | **NeuroLoc-Δ** |
|---|---|---|
| Dominio biologico | Tutti gli eucarioti | Solo cervello umano |
| Numero di classi | 9 | **14 (9 + 5 neuronali)** |
| Classi neuronali | ❌ | ✅ Assone, Dendrite, Pre/Post-sinapsi, Mielina |
| Encoder di sequenza | ProtT5 (1 encoder) | **ESM-2 + ProtT5 (2 encoder)** |
| Testo funzionale | ❌ | ✅ PubMedBERT su `card_protein` |
| Struttura 3D | ❌ | ✅ AlphaFold (15 descrittori) |
| Fonti di training | DeepLoc dataset | **11 fonti** (DeepLoc, UniProt, HPA, OpenCell, LOPIT×3, BioID, SynGO, MitoCarta) |
| Canale di rumore | ❌ | ✅ Sensibilità/specificità appresi per fonte |
| Analisi varianti | ❌ | ✅ NeuroLoc-Δ (meccanismo molecolare rotto) |
| Percentuali quantitative | ❌ | ✅ Frazioni citosolica/nucleare calibrate |
| Filtro sangue/cervello | ❌ | ✅ Esclusione proteine plasmatiche epatiche |
| Metrica di selezione | Spazio proteico generale | **Solo proteine umane cerebrali** |

### Metriche di performance — benchmark standard (test HPA, 9 classi generali)

| Modello | Accuracy | Jaccard | Micro F1 | Macro F1 |
|---|---|---|---|---|
| DeepLoc 2.0 (ESM-1b) | 0.34 | 0.48 | 0.57 | 0.44 |
| DeepLoc 2.0 (ProtT5) — SOTA | 0.39 | **0.53** | 0.60 | 0.46 |
| NeuroLoc-Δ v1 (baseline) | — | 0.48–0.50 | — | — |
| NeuroLoc-Δ v2 (in corso) | — | target ≥ 0.53 | — | — |

*Valori DeepLoc 2.0 da Thumuluri et al. 2022, Tab. 2. Valori NeuroLoc-Δ da validazione interna out-of-fold.*

### Commento critico

**Vantaggi rispetto allo stato dell'arte:**
Le 5 classi neuronali non hanno confronto diretto: nessun tool esistente le predice. Analogamente, l'analisi delle varianti (NeuroLoc-Δ), le percentuali quantitative di ripartizione e l'integrazione multi-fonte con canale di rumore sono contributi originali senza equivalenti pubblicati per il dominio cerebrale.

**Limitazioni oneste:**
Sul benchmark standard (9 classi generali, test HPA indipendente), la versione 1 del modello è alla pari con DeepLoc 2.0 ESM-1b ma sotto la versione ProtT5. La versione 2 mira a colmare questo gap attraverso la fusione dei due encoder e i contributi multimodali, ma il dataset di training — pur esteso con fonti aggiuntive — rimane più piccolo rispetto al dataset globale di DeepLoc. I numeri assoluti di accuracy sul benchmark generico potrebbero non superare lo stato dell'arte; il valore scientifico principale del progetto è la **granularità neuronale** e la **specializzazione di dominio**.

---

## 📁 Struttura del repository

```
Brain-proteins-miss-localization/
│
├── 01_dati.ipynb                  # Notebook 1: acquisizione, pulizia e strutturazione dei dati
│                                  # Download fonti, costruzione osservazioni multi-fonte,
│                                  # filtro cerebrale/sangue, card_protein, frazioni, grafo
│
├── 02_embedding_modello_v2.ipynb  # Notebook 2: encoding, modello e predizioni
│                                  # Hardware detection adattivo, embedding ESM-2/ProtT5/
│                                  # PubMedBERT/AlphaFold, training multi-task, ensemble,
│                                  # calibrazione, analisi varianti
│
├── schema_workflow.html           # Schema visivo interattivo del workflow completo
│                                  # (annotazioni progettuali per ogni passo)
│
├── neuroloc_output.html           # Output di esempio per 5 proteine reali
│                                  # (Tau, alfa-sinucleina, SHANK3, APP, Neurexina-1α)
│                                  # con grafici interattivi e heatmap comparativa
│
└── README.md                      # Questo file
```

**Cartelle generate a runtime** (non incluse nel repository):

```
cache/          # File scaricati dalle fonti (DeepLoc, UniProt, HPA, ClinVar, ...)
data/           # Artefatti prodotti dal notebook 01 (proteine.parquet, osservazioni.parquet, ...)
data/emb_*/     # Embedding calcolati dai PLM (file grandi, ripartibili)
data/alphafold/ # Strutture CIF scaricate da AlphaFold DB
```

---

## ⚙️ Come usare il progetto

### Prerequisiti

```
Python >= 3.11
CUDA >= 11.8 (opzionale ma fortemente consigliato; la CPU è supportata)
RAM >= 16 GB
VRAM >= 6 GB (per ESM-2 650M e ProtT5-XL in sequenza)
```

### Installazione delle dipendenze

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install transformers datasets pandas numpy scikit-learn
pip install pyarrow fastparquet openpyxl
pip install requests urllib3 tqdm sentencepiece
pip install matplotlib scipy psutil
```

### Esecuzione

Il progetto è strutturato in due notebook Jupyter da eseguire in sequenza.

**Passo 1 — Preparazione dei dati**

```bash
jupyter notebook 01_dati.ipynb
```

Eseguire tutte le celle nell'ordine. Il notebook scarica automaticamente tutte le fonti necessarie (≈ 2–4 GB totali) nella cartella `cache/` e produce i file Parquet in `data/`. Il download viene saltato per i file già presenti in cache.

> **Nota:** MitoCarta 3.0 richiede download manuale da [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta) a causa di restrizioni di accesso automatico. Salvare il file come `cache/mitocarta3.xlsx`. Il notebook funziona anche senza questo file (la fonte viene saltata con un avviso).

**Passo 2 — Encoding e modello**

```bash
jupyter notebook 02_embedding_modello_v2.ipynb
```

Eseguire prima la cella 0 (hardware detection) poi le celle nell'ordine. Il calcolo degli embedding è ripartibile: se il notebook viene interrotto, alla ripartenza riparte dal punto in cui si era fermato. I tempi indicativi su GTX 1080 (8 GB):

| Fase | Tempo |
|---|---|
| Embedding ESM-2 650M (19.044 sequenze) | ≈ 230 min |
| Embedding ProtT5-XL | ≈ 290 min |
| Embedding PubMedBERT | ≈ 15 min |
| Training rete (ricerca + CV completa) | ≈ 30–40 min |

**Solo CPU:** impostare `ESM = 'facebook/esm2_t30_150M_UR50D'`, `USE_T5 = False`, `N_SCREEN = 8`.

### Output

Al termine del notebook 02 sono disponibili:

| File | Contenuto |
|---|---|
| `data/probabilita_proteine_cervello.xlsx` | Probabilità calibrate per 14 compartimenti, ripartizione quantitativa e card_protein per ogni proteina cerebrale |
| `data/varianti_mislocalizzazione.xlsx` | Score di mislocalizzazione e meccanismo molecolare per ogni variante ClinVar |
| `data/confronto_modelli.xlsx` | Metriche di tutti i modelli sul test HPA indipendente |
| `data/roc_confronto.png` | Curve ROC per compartimento e confronto con DeepLoc 2.0 |

---

## 📚 Riferimenti

**Stato dell'arte — DeepLoc 2.0**
> Thumuluri, V., Almagro Armenteros, J.J., Johansen, A.R., Nielsen, H., Winther, O. (2022).
> *DeepLoc 2.0: multi-label subcellular localization prediction using protein language models.*
> Nucleic Acids Research, 50(W1), W228–W234. https://doi.org/10.1093/nar/gkac278

**Encoder proteico — ESM-2**
> Lin, Z., Akin, H., Rao, R., et al. (2023).
> *Evolutionary-scale prediction of atomic-level protein structure with a language model.*
> Science, 379(6637), 1123–1130. https://doi.org/10.1126/science.ade2574

**Encoder proteico — ProtT5**
> Elnaggar, A., Heinzinger, M., Dallago, C., et al. (2022).
> *ProtTrans: Toward Understanding the Language of Life Through Self-Supervised Learning.*
> IEEE Transactions on Pattern Analysis and Machine Intelligence, 44(10), 7112–7127. https://doi.org/10.1109/TPAMI.2021.3095381

**Encoder testuale — PubMedBERT**
> Gu, Y., Tinn, R., Cheng, H., et al. (2021).
> *Domain-specific language model pretraining for biomedical natural language processing.*
> ACM Transactions on Computing for Healthcare, 3(1), 1–23. https://doi.org/10.1145/3458754

**Struttura 3D — AlphaFold**
> Jumper, J., Evans, R., Pritzel, A., et al. (2021).
> *Highly accurate protein structure prediction with AlphaFold.*
> Nature, 596, 583–589. https://doi.org/10.1038/s41586-021-03819-2

**Dati di frazionamento cellulare — Hein 2024**
> Hein, M.Y., et al. (2024).
> *A human interactome in three quantitative dimensions organized by stoichiometries and abundances.*
> Cell. https://doi.org/10.1016/j.cell.2015.09.053

**Database sinaptico — SynGO**
> Koopmans, F., van Nierop, P., Andres-Alonso, M., et al. (2019).
> *SynGO: An Evidence-Based, Expert-Curated Knowledge Base for the Synapse.*
> Neuron, 103(2), 217–234. https://doi.org/10.1016/j.neuron.2019.05.002

**Proteomica spaziale — LOPIT**
> Mulvey, C.M., et al. (2021).
> *Spatiotemporal proteomic profiling of the pro-inflammatory response to lipopolysaccharide in the THP-1 macrophage cell line.*
> Nature Communications, 12, 5773. https://doi.org/10.1038/s41467-021-26000-9

---

## Licenza

Questo progetto è rilasciato sotto licenza MIT. Vedere il file [LICENSE](LICENSE) per i dettagli.

---

*NeuroLoc-Δ — Università di Salerno · Ricerca accademica*
