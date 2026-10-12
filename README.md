# 🧠 Brain Protein Mislocalization — Localizzazione Subcellulare Cerebrale

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-2.1%2B-EE4C2C?logo=pytorch&logoColor=white)
![License](https://img.shields.io/badge/Licenza-MIT-green)
![Status](https://img.shields.io/badge/Stato-In%20sviluppo-yellow)

Predizione probabilistica della **localizzazione subcellulare** di proteine umane espresse nel cervello, con un modello multi-modale che integra sequenza aminoacidica, testo funzionale, struttura 3D AlphaFold e annotazioni biologiche da 13 fonti sperimentali.

---

## Descrizione

Le malattie neurodegenerative (Alzheimer, Parkinson, SLA) sono spesso associate alla **mislocalization proteica**: proteine che si ritrovano in compartimenti cellulari anomali, dove formano aggregati tossici o perdono la loro funzione fisiologica. Predire *dove* una proteina cerebrale umana si localizza — e con quale probabilità — è il primo passo per capire dove il processo può andare storto.

Questo progetto costruisce una pipeline end-to-end che:

1. Raccoglie ed integra etichette di localizzazione da **13 fonti biologiche** (UniProt, HPA, DeepLoc, OpenCell, LOPIT, BioID, MitoCarta, SynGO, GOA e altre)
2. Addestra **MM-CAN-SubCell** (*Multi-Modal Cross-Attention Network for Subcellular Localization*), un modello PyTorch che fonde embedding ESM-2, BioLinkBERT e feature strutturali AlphaFold tramite cross-attention
3. Predice una **distribuzione di probabilità** su 14 compartimenti (9 organelli + 5 domini neuronali) per ogni proteina della coorte cerebrale umana

---

## Struttura della repository

```
Brain-miss-localization/
│
├── Try-01/
│   ├── 01_dati_cervello_umano.ipynb   # Pipeline dati: download, preprocessing, etichette
│   └── 02_modeling.ipynb              # Architettura MM-CAN-SubCell, training, valutazione
│
├── Documentazione/
│   ├── schema_01_dati.html            # Schema interattivo workflow dati
│   └── schema_02_modeling.html        # Schema interattivo workflow modeling
│
├── data/                              # Generata da 01_dati (non inclusa nel repo)
│   ├── proteine.parquet               # Coorte cerebrale + feature + card_protein
│   ├── osservazioni.parquet           # Tabella lunga proteina × fonte × classe
│   ├── config_dati.json               # Metadati classi e fonti
│   ├── embeddings/                    # Embedding ESM-2 e BioLinkBERT (cache NPY)
│   └── eda_classi.png                 # Distribuzione positivi per classe
│
├── cache/                             # File scaricati automaticamente da 01_dati
│   ├── alphafold/                     # Strutture PDB per proteina (cache locale)
│   ├── mitocarta3.xlsx                # Download manuale (Broad Institute)
│   └── ...
│
├── models/                            # Checkpoint PyTorch (generati da 02_modeling)
│   ├── mm_can_subcell_best.pt
│   └── baselines/
│
├── figures/                           # Figure di valutazione (generate da 02_modeling)
│   ├── roc_comparison_final.png
│   ├── umap_mmcan_subcell.png
│   └── confusion_matrix_mmcan.png
│
├── results/
│   └── metrics_comparison.csv        # Tabella comparativa tutti i modelli
│
├── requirements.txt
└── README.md
```

---

## Workflow — Pipeline dei Dati (`01_dati_cervello_umano.ipynb`)

Il notebook costruisce la coorte cerebrale umana a partire da zero, scaricando e integrando 13+ fonti biologiche. L'output è un insieme di file Parquet pronti per il modeling.

### Schema interattivo

> 📊 **[Apri lo schema completo del workflow dati](Documentazione/schema_01_dati.html)**
>
> Lo schema illustra tutti gli 11 step del notebook: configurazione, download fonti, costruzione osservazioni multi-fonte, GOA sperimentale, funnel dei filtri, feature AlphaFold, `card_protein` e salvataggio.

### Passaggi principali

#### 1. Configurazione e download fonti

- Definizione delle **14 classi target**: 9 organelli (Asse A, distribuzione normalizzata) + 5 domini neuronali (Asse B, probabilità indipendenti)
- Download automatico con retry su errori HTTP (429/5xx) e scrittura atomica (`.part` → rinomina)
- Fonti opzionali (GOA, SynGO, fonti esterne CSV) non bloccano l'esecuzione se assenti
- **Download manuale richiesto**: `cache/mitocarta3.xlsx` da [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta)

#### 2. Universo proteico: UniProt + DeepLoc filtrato

- UniProt reviewed, `organism:9606`, non-fragment → **20.430 proteine**
- DeepLoc 2.0 (28.303 voci) filtrato per sequenza identica a UniProt umano → **6.755 umane** (le specie non umane escono: domini diversi di curatela)
- Costruzione del dizionario `simbolo genico → UniProt` (ufficiale + alias non ambigui) e `sequenza → UniProt`
- Test HPA DeepLoc (1.717 proteine) **isolato immediatamente** come set di test indipendente: non entra mai nelle osservazioni di addestramento

#### 3. Osservazioni multi-fonte → tabella lunga

Per ogni fonte viene costruita una tabella `(proteina, fonte, classe) → 1/0` rispettando la **regola delle assenze**: una fonte produce `0` solo per le proteine che riporta e per le classi che sa misurare.

| Fonte | Classi coperte | Tipo di evidenza |
|---|---|---|
| UniProt forte | tutti gli organelli + domini neuronali | ECO sperimentale (ECO:0000269, ECO:0000314…) |
| UniProt debole | tutti gli organelli + domini neuronali | inferenza / similarità |
| HPA alta / approved | 8 organelli (no extracellulare) | microscopia IHC |
| MitoCarta 3.0 | solo Mitocondri | inventario completo → zeri affidabili |
| OpenCell | 8 organelli | live cell imaging HEK293T |
| SynGO | Presinapsi / Postsinapsi | curato manuale |
| GOA sperimentale | tutti (14 classi) | solo positivi (mondo aperto, mai zeri) |
| LOPIT ×3 | 8 organelli | SVM su profili di frazionamento (p ≥ 0.7) |
| BioID | 8 organelli | prossimità APEX/BioID NMF |
| DeepLoc (umane) | 9 organelli | dataset oro |
| Fonti esterne CSV | Assone / Dendrite / Presinapsi… | proteomiche iPSC (manuali) |

> **GOA sperimentale** è la novità principale rispetto alla versione precedente: propaga le annotazioni lungo il grafo ontologico (`is_a` + `part_of`), esclude esosomi (GO:0070062) e porta 53+ proteine assonali, 67 dendritiche, 47 presinaptiche — il minimo critico per apprendere segnali di targeting specifici nei domini neuronali.

#### 4. Espressione cerebrale ed esclusione del sangue

- **HPA RNA tessuto**: 15 regioni cerebrali, soglia `nTPM ≥ 10`
- **HPA single-cell**: neuroni, astrociti, oligodendrociti, microglia, soglia `nCPM ≥ 5`
- **Sangue escluso dalla coorte**: secretoma epatico + cellule ematiche arricchite senza espressione cerebrale + emoglobine (regex `HB*`) → **2.781 proteine rimosse**
- Quote di frazionamento citosolico/nucleare da Hein 2024 (HEK293T) e Itzhak 2016 (HeLa): 9.438 proteine (nota: linee tumorali, non neuroni)

#### 5. Funnel dei filtri → coorte finale

```
UniProt umane reviewed          20.430
→ lunghezza 41–12.000 aa        20.358
→ senza sangue                  17.649
→ espresse nel cervello         13.298   ← coorte finale
  di cui addestrabili (≥1 obs)  13.043
  di cui test HPA indipendente     255
```

Validazione con `assert`: nessun UniProt duplicato, sequenze solo AA canonici, nessun leak test→train, nessuna proteina del sangue residua, `obs ∈ {0.0, 1.0}`.

#### 6. Feature strutturali AlphaFold (23 colonne `af_*`)

Scaricate tramite API `alphafold.ebi.ac.uk` con 8 worker paralleli e cache locale (PDB per proteina). Le feature includono:

- pLDDT medio, regionale (N/centro/C), per terminale C (ultimi 5 residui)
- Frazioni di confidenza (very low / low / confident / very high)
- Densità di contatto Cα, esposizione N10/C5, raggio di girazione normalizzato
- Frazioni elica/foglietto, lunghezza massima regione disordinata, conteggio IDR ≥ 30 residui
- Eliche lunghe idrofobiche (proxy TM), patch di lisine esposte (proxy M6P), idrofobiche superficiali
- Flag `af_partial` (struttura con copertura < 90%)

#### 7. `card_protein` — profilo testuale esperto

Sei sezioni estratte dai campi UniProt per ogni proteina:

1. Identità & funzione (nome, famiglia, attività catalitica, via metabolica)
2. Domini, motivi, ripetizioni, zinc finger, coiled-coil (campi FT)
3. Malattie neurologiche associate (17 pattern classificatori: Alzheimer, Parkinson, SLA…)
4. Regolazione, PTM, partner/complessi
5. Espressione cerebrale (HPA tessuto + single-cell + specificità tissutale UniProt)
6. Profilo esperto a regole: classe funzionale (20 categorie) + rilevanza neurologica + struttura AlphaFold

> **Anti-leakage**: `card_protein` rimuove tutte le frasi che nominano un compartimento cellulare, un segnale di targeting o una modificazione lipidica (regex su 40+ pattern). `card_full` conserva il testo integrale ed è usata nel notebook 02 per misurare il delta di prestazione dovuto a eventuale leakage.

#### Output del notebook 01

| File | Contenuto |
|---|---|
| `data/proteine.parquet` | 13.298 proteine · pid, uniprot_id, gene, sequence, role, has\_label, is\_brain/neuronal/oligo, frac\_cyto/nuc, card\_protein/full, af\_\*, aux\_\*, ctx\_\* |
| `data/osservazioni.parquet` | 439.719 righe · pid, uniprot\_id, fonte, classe, obs (0.0/1.0) |
| `data/config_dati.json` | Metadati: classi, fonti positive, organelli, domini neuronali |
| `data/eda_classi.png` | Distribuzione positivi per classe e per fonte |

---

## Workflow — Modeling (`02_modeling.ipynb`)

Il notebook addestra **MM-CAN-SubCell** e lo confronta con quattro baseline.

### Schema interattivo

> 📊 **[Apri lo schema completo del workflow di modeling](Documentazione/schema_02_modeling.html)**
>
> Lo schema illustra i 10 step del notebook: setup, soft labels, embedding multi-modali, architettura MM-CAN-SubCell, training, 5-fold CV, confronto baseline, figure ROC, UMAP e output predittivo.

### Architettura MM-CAN-SubCell

Il modello fonde **quattro modalità** con un meccanismo di cross-attention:

```
RAMO A  ESM-2 seq. completa   →  E_seq  ∈ R^D_esm
RAMO B  ESM-2 frammenti N/C/M →  E_frag ∈ R^(3·D_esm)
RAMO C  BioLinkBERT card_prot →  E_text ∈ R^1024
RAMO D  Feature AlphaFold+aux →  F_struct ∈ R^D_struct

        Projection Layer (Linear + LayerNorm + GELU per ogni ramo)
                          ↓
        CROSS-ATTENTION (8 heads)
        Q = H_text  |  K = V = mean(H_seq, H_frag)
        Attended = LayerNorm(attn_out + H_text)    [residual]
                          ↓
        FUSION: Concat([Attended · H_text · H_struct])  →  1088-dim
        Linear(1088→512) + LN + GELU → Linear(512→256) + LN + GELU
                          ↓
        OUTPUT HEAD: Linear(256 → N_CLASSI) → Softmax
        p = [p₁, …, pC]   con  Σpᵢ = 1.0
```

**Giustificazione della cross-attention**: il testo funzionale (Q) interroga selettivamente i pattern di sequenza (K/V) più informativi per ciascun compartimento, catturando correlazioni cross-modali non accessibili ai modelli mono-modali.

### Soft labels — distribuzioni morbide

Le etichette non sono classi discrete ma **distribuzioni di probabilità** costruite come media pesata delle osservazioni multi-fonte:

```
ŷ_c = Σ_f (w_f · obs_{f,c}) / Σ_f (w_f · 𝟙[f misura c])
```

I pesi riflettono la specificità stimata di ogni fonte (DeepLoc = 1.0, SynGO = 1.0, UniProt forte = 0.95, … , UniProt debole = 0.50). Le distribuzioni degli organelli sono normalizzate a somma=1; i domini neuronali sono probabilità indipendenti clippate in [0,1]. Label smoothing ε=0.05 per distribuzioni quasi-one-hot.

### Loss function

```
𝓛 = KL(y ‖ ŷ) + λ · Σ (1−ŷᵢ)^γ · log(ŷᵢ)   [λ=0.1, γ=2]
```

KL divergence per le distribuzioni morbide + componente Focal per gestire le classi minoritarie.

### Training

| Iperparametro | Valore |
|---|---|
| Ottimizzatore | AdamW · lr=3e-4 · weight\_decay=1e-4 |
| Scheduler | Linear warm-up (5 epoche) → CosineAnnealingWarmRestarts (T₀=10) |
| Epoche massime | 80 con early stopping (patience=15 su val\_KL) |
| Batch size effettivo | 128 (32 × gradient accumulation ×4) |
| Regularizzazione | Dropout 0.3 · Stochastic Depth 0.1 · Gradient clipping max\_norm=1.0 |
| Precisione | AMP autocast (float16 su CUDA) |
| Split | Stratificato 70/15/15 per classe primaria degli organelli |
| Cross-validation | 5-fold stratificata |

### Baseline a confronto

| Modello | Input | Note |
|---|---|---|
| Dummy majority | — | lower bound |
| Logistic Regression | ESM + BERT + struct → PCA 256D | solver SAGA, multinomial |
| Random Forest (500 alberi) | PCA 128D | max\_depth=20, class\_weight=balanced |
| MLP semplice (3 hidden layers) | concat flat, senza cross-attention | stessa loss KL+Focal |
| **MM-CAN-SubCell** | 4 rami + cross-attention | modello proposto |

### Figure di output

- `figures/roc_comparison_final.png` — curva ROC macro-averaged obbligatoria (5 modelli sovrapposti)
- `figures/roc_per_compartimento.png` — subplot ROC per ogni compartimento
- `figures/precision_recall.png` — curve Precision-Recall per compartimento
- `figures/confusion_matrix_mmcan.png` — matrice di confusione MM-CAN-SubCell
- `figures/umap_mmcan_subcell.png` — UMAP del penultimo strato 256-dim
- `figures/MM_CAN_SubCell_architecture.png` — schema architetturale
- `figures/training_curve_mmcan.png` — curve di loss train/val per epoca

---

## Requisiti e installazione

### Dipendenze Python

```
Python ≥ 3.10
PyTorch ≥ 2.1.0
fair-esm ≥ 2.0.0
transformers ≥ 4.35.0
scikit-learn ≥ 1.3.0
biopython ≥ 1.81
umap-learn ≥ 0.5.3
pandas
numpy
scipy
pyarrow
requests
matplotlib
seaborn
openpyxl
```

### Installazione

```bash
# Clona la repository
git clone https://github.com/tuonome/Brain-miss-localization.git
cd Brain-miss-localization

# Crea un ambiente virtuale (consigliato)
python -m venv venv
source venv/bin/activate        # Linux/macOS
# oppure: venv\Scripts\activate  # Windows

# Installa le dipendenze
pip install -r requirements.txt
```

> **Nota GPU**: il training richiede una GPU con almeno 8 GB di VRAM (testato su GTX 1080). Su CPU il calcolo degli embedding ESM-2 può richiedere diverse ore. Su Google Colab Pro (T4) il training completo impiega circa 2–3 ore.

---

## Come riprodurre i risultati

Eseguire i notebook **nell'ordine indicato**:

### Step 1 — Download manuale prerequisiti

Prima di avviare il notebook 01, scaricare manualmente:

| File | Fonte | Destinazione |
|---|---|---|
| `Human MitoCarta3.0` (`.xlsx`) | [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta) | `cache/mitocarta3.xlsx` |
| `SynGO ontologies` (`.xlsx`) | [syngoportal.org](https://www.syngoportal.org) | `data/esterni/syngo_ontologies.xlsx` |
| Fonti neuronali opzionali | Supplementi pubblicazioni (vedi notebook) | `data/esterni/fonti/<nome>.csv` |

### Step 2 — Pipeline dati

```bash
jupyter notebook Try-01/01_dati_cervello_umano.ipynb
```

Eseguire tutte le celle in ordine. Il notebook scarica automaticamente tutte le altre fonti, calcola le feature AlphaFold (alcune ore la prima volta, poi cache), costruisce le `card_protein` e salva i file Parquet in `data/`.

### Step 3 — Modeling

```bash
jupyter notebook Try-01/02_modeling.ipynb
```

Eseguire tutte le celle in ordine. Gli embedding ESM-2 e BioLinkBERT vengono calcolati la prima volta e salvati in `data/embeddings/` per i run successivi.

> Tutti i risultati sono riproducibili con `SEED=42` e le versioni di libreria specificate in `requirements.txt`.

---

## Fonti dati

| Fonte | Tipo | URL |
|---|---|---|
| **UniProt** (reviewed, human) | Localizzazione subcellulare + sequenza + annotazioni | [uniprot.org](https://www.uniprot.org) |
| **DeepLoc 2.0** | Dataset oro train/test per localizzazione | [services.healthtech.dtu.dk](https://services.healthtech.dtu.dk/services/DeepLoc-2.0/) |
| **Human Protein Atlas (HPA)** | Microscopia IHC, RNA tessuto, RNA single-cell | [proteinatlas.org](https://www.proteinatlas.org) |
| **OpenCell** | Live cell imaging (HEK293T, tag endogeno) | [opencell.czbiohub.org](https://opencell.czbiohub.org) |
| **LOPIT** (×3) | Proteomica spaziale per frazionamento | [github.com/czbiohub-sf/Organelle\_IP](https://github.com/czbiohub-sf/Organelle_IP_analyses_and_figures) |
| **BioID 2021** | Prossimità APEX2/BioID NMF | idem |
| **MitoCarta 3.0** | Inventario completo proteoma mitocondriale umano | [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta) |
| **SynGO** | Proteine sinaptiche curate manualmente | [syngoportal.org](https://www.syngoportal.org) |
| **Gene Ontology (GOA)** | Annotazioni sperimentali compartimento cellulare | [geneontology.org](https://geneontology.org) |
| **AlphaFold DB** | Strutture 3D predette per proteoma umano | [alphafold.ebi.ac.uk](https://alphafold.ebi.ac.uk) |
| **SynaptoPathyDB** | Geni associati a sinaptopatia | [synaptopathydb.org](https://www.synaptopathydb.org) |
| **Hein et al. 2024** | Frazionamento citosol/nucleo HEK293T | [github.com/czbiohub-sf/Organelle\_IP](https://github.com/czbiohub-sf/Organelle_IP_analyses_and_figures) |
| **Itzhak et al. 2016** | Frazionamento spaziale HeLa (eLife) | idem |

---

## Citazioni principali

- Lin et al. (2023) *Science* — ESM-2: Language models of protein sequences at the scale of evolution enable accurate structure prediction
- Yasunaga et al. (2022) — BioLinkBERT: Pretraining Language Models with Document Links
- Thumuluri et al. (2022) *Nucleic Acids Research* — DeepLoc 2.0: multi-label subcellular localization prediction using protein language models
- Almagro Armenteros et al. (2019) *Nature Methods* — SignalP 5.0 / TargetP 2.0: N-terminal sequence signals
- Geng (2016) *IEEE TNNLS* — Label Distribution Learning
- Hinton et al. (2015) — Distilling the Knowledge in a Neural Network (label smoothing / KL divergence)
- Lin et al. (2017) *ICCV* — Focal Loss for Dense Object Detection

---

## Licenza

Questo progetto è rilasciato sotto licenza **MIT**. Vedi il file [`LICENSE`](LICENSE) per i dettagli.

Le fonti dati utilizzate hanno licenze proprie: UniProt (CC BY 4.0), HPA (CC BY-SA 3.0), OpenCell (BSD), GOA (CC BY 4.0), AlphaFold DB (CC BY 4.0). MitoCarta e SynGO richiedono il download diretto dai rispettivi siti con accettazione dei termini di utilizzo.
