# NeuroLoc-Δ — Localizzazione Subcellulare di Proteine Cerebrali Umane

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-2.1%2B-EE4C2C?logo=pytorch&logoColor=white)
![Status](https://img.shields.io/badge/Stato-In%20sviluppo-yellow)
![License](https://img.shields.io/badge/Licenza-MIT-green)

Predizione probabilistica della **localizzazione subcellulare** di proteine umane espresse nel cervello, con un modello multi-modale che integra sequenza aminoacidica, testo funzionale, struttura 3D AlphaFold e annotazioni biologiche da 13 fonti sperimentali.

---

## Problema e Motivazione

Le malattie neurodegenerative (Alzheimer, Parkinson, SLA, FTD) sono associate alla **mislocalization proteica**: proteine che si accumulano in compartimenti cellulari anomali, dove formano aggregati tossici o perdono la loro funzione fisiologica. Predire *dove* una proteina cerebrale umana si localizza — e con quale probabilità — è il primo passo per identificare dove il processo può andare storto.

I predittori convenzionali (DeepLoc 2.0, TargetP 2.0) hanno tre limitazioni critiche per questo contesto:
- Addestrati su proteomi multi-specie (~75% sequenze non umane), con distribuzione di etichette non rappresentativa del cervello umano
- Agnostici rispetto al tipo cellulare: ignorano la polarità morfologica dei neuroni (assone vs. dendrite) e la specializzazione della mielina
- Output categoriale discreto: non modellano la distribuzione probabilistica di una proteina tra più compartimenti

**NeuroLoc-Δ** è un framework specializzato per la neuroproteomicа di precisione. Opera esclusivamente su *Homo sapiens* (TaxID 9606), filtra le proteine espresse nel cervello, esclude il proteoma ematico, e predice distribuzioni soft su due assi biologicamente distinti:
- **Asse A** — 9 organelli ubiquitari (distribuzione, somma a 1.0)
- **Asse B** — 5 domini neuronali non esclusivi: Assone, Dendrite, Presinapsi, Postsinapsi, Mielina (probabilità indipendenti in [0,1])

---

## Architettura: MM-CAN-SubCell v2.0

Il modello fonde **quattro modalità** con un meccanismo di cross-attention testuale:

```
RAMO A  ESM-2 650M (sequenza completa)    →  E_seq  ∈ R^1280
RAMO B  ESM-2 650M (frammenti N/centro/C) →  E_frag ∈ R^3840
RAMO C  BioLinkBERT-large (card_protein)  →  E_text ∈ R^1024
RAMO D  Feature biofisiche AlphaFold 3D   →  F_struct ∈ R^(STRUCT_DIM+1)

        ↓  Projection Layer (Linear + LayerNorm + GELU per ogni ramo)

        CROSS-ATTENTION (8 teste)
        Q = H_text  |  K = V = mean(H_seq, H_frag)
        Attended = LayerNorm(attn_out + H_text)     [residual connection]

        ↓  FUSION: Concat([Attended, H_text, H_struct])  →  1088-dim
           Linear(1088→512) + LN + GELU → Linear(512→256) + LN + GELU

        ↓  DOUBLE-HEAD OUTPUT (v2.0)
           head_org: Linear(256→9)  → Softmax       [Asse A, Σp=1]
           head_neu: Linear(256→5)  → Sigmoid        [Asse B, p∈[0,1]]
```

**Novità v2.0 rispetto all'architettura originale:**

| Componente | Versione Precedente | v2.0 |
|---|---|---|
| Output head | Singolo head, Softmax su 14 classi | Double-Head separato per organelli e domini neuronali |
| Loss | KL+Focal unificata su 14 classi | KLFocalLoss (Asse A) + AsymmetricLoss (Asse B) + MultiTaskUncertaintyLoss |
| Bilanciamento task | α=0.5 statico | Incertezza Omoscedastica apprendibile (Kendall et al., 2018) |
| Robustezza card assente | Nessuna | Card-Dropout (p=0.35) + Dynamic Text Gate in inferenza |
| AlphaFold missing | Zeri non distinguibili | Flag binario `af_missing_mask` concatenato a F_struct |
| Cross-validation | StratifiedKFold (fallisce su Mielina N=4) | MultilabelStratifiedKFold (Sechidis et al., 2011) |

---

## Struttura della Repository

```
Brain-proteins-miss-localization/
│
├── try_01/
│   ├── 01_data_brain.ipynb        # Pipeline dati: download, preprocessing, etichette
│   └── 02_modeling.ipynb          # Architettura MM-CAN-SubCell v2.0, training, valutazione
│
├── data/                          # Generata da 01_data_brain (non versionata)
│   ├── proteine.parquet           # 13.298 proteine con feature, card_protein, card_full, card_degraded
│   ├── osservazioni.parquet       # Tabella lunga proteina × fonte × classe
│   └── config_dati.json           # Metadati classi e fonti
│
├── cache/                         # File scaricati automaticamente
│   ├── alphafold/                 # Strutture PDB per proteina (cache locale)
│   └── mitocarta3.xlsx            # Download manuale (Broad Institute)
│
├── models/                        # Checkpoint PyTorch (generati da 02_modeling)
│   ├── mmcan_subcell_robust.pt    # Teacher model (MM-CAN-SubCell v2.0)
│   ├── mmcan_student.pt           # Student model (Sequenza + AlphaFold only)
│   └── baselines/
│
├── neuroloc-delta-improvement-plan.md   # Piano di miglioramento v2.0 con analisi critica
├── requirements.txt
└── README.md
```

---

## Workflow

### Step 1 — Pipeline dei Dati (`try_01/01_data_brain.ipynb`)

Costruisce la coorte cerebrale umana integrando 13 fonti biologiche.

**Download manuale prerequisiti** (prima di avviare il notebook):

| File | Fonte | Destinazione |
|---|---|---|
| `Human MitoCarta3.0` (`.xlsx`) | [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta) | `cache/mitocarta3.xlsx` |
| `SynGO ontologies` (`.xlsx`) | [syngoportal.org](https://www.syngoportal.org) | `data/esterni/syngo_ontologies.xlsx` |

**Funnel della coorte:**

```
UniProt umane reviewed          20.430
→ lunghezza 41–12.000 aa        20.358
→ filtro No-Blood               17.649   (−2.781 proteine ematiche/plasmatiche)
→ espresse nel cervello         13.298   ← coorte finale
  di cui addestrabili (≥1 obs)  13.043
  di cui test HPA indipendente     255
```

**Fonti di etichettatura:**

| Fonte | Classi coperte | Nota |
|---|---|---|
| UniProt forte / debole | Tutti i compartimenti | ECO sperimentale / inferenza |
| HPA (alta / approved) | 8 organelli | IHC, no extracellulare |
| MitoCarta 3.0 | Solo mitocondri | Zeri affidabili su tutto il genoma |
| OpenCell (HEK293T) | 8 organelli | Live cell imaging |
| SynGO | Presinapsi / Postsinapsi | Curatela manuale, peso 1.0 |
| GOA sperimentale | 14 classi | Solo positivi; unica fonte per Mielina (4 positivi) |
| LOPIT ×3 (U2OS/HeLa) | 8 organelli | Frazionamento da linee tumorali — peso 0 per Asse B |
| BioID | 8 organelli | Prossimità APEX/BioID |
| DeepLoc (umane) | 9 organelli | Dataset oro |

**`card_protein` — profilo testuale:**  
Sei sezioni estratte dai campi UniProt (funzione, domini, malattie neurologiche, PTM, espressione cellulare, profilo esperto a regole). Il filtro regex `LOC_RE` rimuove 100+ termini che nominano un compartimento o un segnale di targeting per prevenire il data leakage. Vengono prodotte tre versioni:

- `card_protein` — versione de-leaked (input del modello)
- `card_full` — testo integrale (upper bound del leakage)
- `card_degraded` — anonimizzata anche dai nomi di famiglie proteiche con alias di compartimento (misura del proxy leakage indiretto)

**Output:** `data/proteine.parquet`, `data/osservazioni.parquet`, `data/config_dati.json`

---

### Step 2 — Modeling (`try_01/02_modeling.ipynb`)

Addestra MM-CAN-SubCell v2.0 e i modelli di confronto.

**Soft labels:** ogni proteina riceve una distribuzione di probabilità costruita come media pesata delle osservazioni multi-fonte con pesi `PESI_ORG` (per Asse A) e `PESI_NEU` (per Asse B). Le sorgenti da linee tumorali (LOPIT, OpenCell, Hein, Itzhak) hanno peso 0 in `PESI_NEU`.

**Loss:**

```
L_total = MultiTaskUncertaintyLoss(L_org, L_neu)

  L_org = KLFocalLoss(logits_org, y_org)        [KL + λ·Focal, λ=0.1, γ=2.0]
  L_neu = AsymmetricLoss(logits_neu, y_neu)      [γ_neg=4, γ_pos=1, clip=0.05]

  MultiTaskUncertaintyLoss: L = exp(-s₁)·L_org + s₁ + exp(-s₂)·L_neu + s₂
  con s₁, s₂ parametri apprendibili (Kendall et al., 2018)
```

**Training:**

| Parametro | Valore |
|---|---|
| Ottimizzatore | AdamW · lr=3e-4 · weight_decay=1e-4 |
| Scheduler | Warmup lineare (5 epoche) → CosineAnnealingWarmRestarts |
| Epoche | Max 80 con early stopping patience=15 (val KL) |
| Batch size | 32 con gradient accumulation ×4 |
| Cross-validation | MultilabelStratifiedKFold, 5 fold |
| Metriche Asse A | AUC-ROC macro, KL divergence |
| Metriche Asse B | AUPRC, MCC, AUC-ROC per classe |

**Modelli di confronto:** DummyClassifier, Logistic Regression (PCA 256d), Random Forest (500 alberi), MLP semplice (senza cross-attention), MM-CAN-SubCell v2.0

**Protocolli di valutazione aggiuntivi:**

- **Ablation Card (3 condizioni):** training separato con `card_full`, `card_protein`, `card_degraded` + Indice di Leakage Semantico `I_leak`
- **Card-Free evaluation:** inferenza con `E_text = null_card_emb` per simulare proteine orfane
- **Temperature Scaling:** calibrazione post-hoc sui logits dell'Asse A (Guo et al., 2017)
- **SupCon Loss:** loss ausiliaria sul repr_256 per rinforzare i cluster delle classi rare (Khosla et al., 2020)
- **Teacher-Student KD:** distillazione del Teacher multi-modale verso uno Student (Sequenza + AlphaFold) per screening proteome-wide

---

## Installazione

```bash
git clone https://github.com/Cesare-Red/Brain-proteins-miss-localization.git
cd Brain-proteins-miss-localization

python -m venv venv
source venv/bin/activate          # Linux/macOS
# oppure: venv\Scripts\activate   # Windows

pip install -r requirements.txt
```

**Dipendenze principali:**

```
Python ≥ 3.10
torch ≥ 2.1
fair-esm ≥ 2.0
transformers ≥ 4.35
scikit-learn ≥ 1.3
iterative-stratification ≥ 0.1.7
biopython ≥ 1.81
umap-learn ≥ 0.5.3
pandas · numpy · scipy · pyarrow · matplotlib · seaborn · openpyxl
```

> **GPU:** il training richiede almeno 8 GB di VRAM (testato su RTX 3080). Gli embedding ESM-2 su CPU richiedono diverse ore. Su Google Colab Pro (T4) il training completo è circa 2–3 ore.

---

## Fonti Dati

| Fonte | Tipo | URL |
|---|---|---|
| UniProt (reviewed, human) | Sequenza + annotazioni + localizzazione | [uniprot.org](https://www.uniprot.org) |
| DeepLoc 2.0 | Dataset oro per localizzazione | [services.healthtech.dtu.dk](https://services.healthtech.dtu.dk/services/DeepLoc-2.0/) |
| Human Protein Atlas | IHC, RNA tessuto, RNA single-cell | [proteinatlas.org](https://www.proteinatlas.org) |
| OpenCell | Live cell imaging HEK293T | [opencell.czbiohub.org](https://opencell.czbiohub.org) |
| LOPIT ×3 + BioID (Hein 2024) | Proteomica spaziale | [github.com/czbiohub-sf/Organelle_IP](https://github.com/czbiohub-sf/Organelle_IP_analyses_and_figures) |
| Itzhak et al. 2016 | Frazionamento HeLa (eLife) | idem |
| MitoCarta 3.0 | Inventario proteoma mitocondriale | [broadinstitute.org/mitocarta](https://www.broadinstitute.org/mitocarta) |
| SynGO | Proteine sinaptiche curate | [syngoportal.org](https://www.syngoportal.org) |
| Gene Ontology (GOA) | Annotazioni sperimentali CC | [geneontology.org](https://geneontology.org) |
| AlphaFold DB | Strutture 3D predette | [alphafold.ebi.ac.uk](https://alphafold.ebi.ac.uk) |

---

## Piano di Miglioramento

Il file [`neuroloc-delta-improvement-plan.md`](neuroloc-delta-improvement-plan.md) documenta il piano di sviluppo verso la versione v2.0 del sistema, con analisi critica di ogni sub-task. Le principali aree di intervento:

| Priorità | Sub-Task | Descrizione |
|---|---|---|
| 🔴 Critica | ST2 | Sostituire `MMCANSubCell` con `MMCANSubCellRobust` (Double-Head, ASL, Card-Dropout, MultiTaskUncertaintyLoss) |
| 🔴 Critica | ST6 → ST2 | Stratificare `PESI_FONTE` in `PESI_ORG` / `PESI_NEU` prima del training |
| 🟠 Alta | ST6a | Aggiungere SupCon Loss sul repr_256 con Balanced Batch Sampler per le classi rare |
| 🟠 Alta | ST2 | Silver-Standard Prior Expansion per Mielina (N=4) + MultilabelStratifiedKFold |
| 🟡 Media | ST3 | Generare `card_degraded` e calcolare Indice di Leakage Semantico |
| 🟡 Media | ST4 | Protocollo Card-Free + Dynamic Text Gate per inferenza su proteine orfane |
| 🟡 Media | ST6b | Teacher-Student Knowledge Distillation per screening proteome-wide |
| 🟢 Bassa | ST5 | Temperature Scaling (calibrazione post-hoc) sulla testa organelli |

---

## Citazioni

- Lin et al. (2023) *Science* — ESM-2: Language models of protein sequences at the scale of evolution
- Yasunaga et al. (2022) — BioLinkBERT: Pretraining Language Models with Document Links
- Thumuluri et al. (2022) *Nucleic Acids Research* — DeepLoc 2.0: multi-label subcellular localization prediction
- Ben-Baruch et al. (2020) — Asymmetric Loss For Multi-Label Classification
- Kendall & Gal (2018) *NeurIPS* — Multi-Task Learning Using Uncertainty to Weigh Losses
- Khosla et al. (2020) *NeurIPS* — Supervised Contrastive Learning
- Hinton et al. (2015) — Distilling the Knowledge in a Neural Network
- Lin et al. (2017) *ICCV* — Focal Loss for Dense Object Detection
- Sechidis et al. (2011) *ECML* — On the Stratification of Multi-Label Data
- Guo et al. (2017) *ICML* — On Calibration of Modern Neural Networks
- Khosla et al. (2020) — Class-Balanced Loss Based on Effective Number of Samples

---

## Licenza

Rilasciato sotto licenza **MIT**. Le fonti dati hanno licenze proprie: UniProt (CC BY 4.0), HPA (CC BY-SA 3.0), OpenCell (BSD), GOA (CC BY 4.0), AlphaFold DB (CC BY 4.0). MitoCarta e SynGO richiedono accettazione dei termini d'uso dai rispettivi siti.

---

## Schemi Interattivi

Visualizzazioni HTML dei due workflow principali del progetto. Aprirle direttamente nel browser per navigare i passaggi in modo interattivo.

### 📊 Workflow Dati — `01_data_brain.ipynb`

> [**schema-workflow-01-dati-cervello-umano.html**](try_01/schema-workflow-01-dati-cervello-umano.html)
>
> Illustra gli 11 step del notebook: configurazione, download fonti, costruzione osservazioni multi-fonte, GOA sperimentale, funnel dei filtri (20.430 → 13.298 proteine), feature AlphaFold 3D, costruzione `card_protein` / `card_full` / `card_degraded` e salvataggio dei Parquet.

### 📊 Workflow Modeling — `02_modeling.ipynb`

> [**schema-workflow-02-modeling-mm-can-subcell.html**](try_01/schema-workflow-02-modeling-mm-can-subcell.html)
>
> Illustra i 10 step del notebook: setup, soft labels multi-fonte, embedding ESM-2 e BioLinkBERT, architettura MM-CAN-SubCell v2.0 (Double-Head, Card-Dropout, MultiTaskUncertaintyLoss), training con MultilabelStratifiedKFold, protocolli di ablation (Card-Free, Temperature Scaling, SupCon), confronto baseline e figure ROC/UMAP.

