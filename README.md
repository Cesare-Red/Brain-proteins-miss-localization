# NeuroLoc-Δ

**Predizione della localizzazione subcellulare delle proteine cerebrali e dei meccanismi di mislocalizzazione delle varianti missense, con embedding ESM-2 e supervisione multi-fonte.**

Il progetto è composto da **due notebook**:

| Notebook | Cosa fa |
|---|---|
| `01_dati.ipynb` | Scarica, pulisce e unisce tutte le sorgenti. Produce le proteine, le osservazioni di localizzazione per fonte, i target ausiliari, le varianti. |
| `02_embedding_modello.ipynb` | Calcola gli embedding ESM-2 e addestra il modello con grid search e cross-validation per omologia. Si confronta con DeepLoc 2.0 sul test indipendente HPA, valuta il sottoinsieme cerebrale e assegna un punteggio di mislocalizzazione alle varianti ClinVar. |

---

## Indice

1. [Obiettivo](#obiettivo)
2. [Idee metodologiche](#idee-metodologiche)
3. [Struttura del repository](#struttura-del-repository)
4. [Installazione](#installazione)
5. [Guida rapida](#guida-rapida)
6. [Dati scaricati automaticamente](#dati-scaricati-automaticamente)
7. [Dati da scaricare a mano](#dati-da-scaricare-a-mano)
8. [Cosa succede nei due notebook](#cosa-succede-nei-due-notebook)
9. [Output](#output)
10. [Valutazione e confronto con lo stato dell'arte](#valutazione-e-confronto-con-lo-stato-dellarte)
11. [Test interni](#test-interni)
12. [Limiti noti](#limiti-noti)
13. [Riferimenti](#riferimenti)

---

## Obiettivo

Le domande sono due:

1. **Dove dovrebbe stare una proteina espressa nel cervello?** La risposta è una probabilità per ciascuno dei 9 compartimenti di DeepLoc 2.0 e per 4 compartimenti neuronali (assone, dendrite, presinapsi, postsinapsi).
2. **Una variante missense sposta la proteina? E con quale meccanismo?** I meccanismi considerati sono la rottura del segnale di sorting, la perdita di inserzione in membrana e la destabilizzazione.

## Idee metodologiche

| # | Problema | Soluzione adottata |
|---|---|---|
| 1 | Le banche dati di localizzazione hanno affidabilità diverse (UniProt incompleto, HPA Approved meno solido, mappe neuronali di topo). | **Canale di rumore per fonte.** Il modello stima la localizzazione latente e osserva ogni fonte con una sensibilità e una specificità per classe, apprese dai dati e inizializzate dove la fonte si sovrappone a DeepLoc. DeepLoc è l'ancora sperimentale. |
| 2 | Un compartimento non annotato non è necessariamente assente. | La sensibilità per fonte assorbe l'incompletezza. Per ogni fonte, un'assenza conta solo se la fonte sa misurare quel compartimento. |
| 3 | I neuroni hanno compartimenti che le linee cellulari non hanno. | **Testa neuronale** separata (assone, dendrite, pre/post-sinapsi), addestrata solo sulle proteine espresse nei neuroni (HPA single-cell). |
| 4 | Le feature annotate (PTM, NLS, TM...) contengono la risposta o non cambiano con la mutazione. | Diventano **target ausiliari multi-task**: il modello le deve prevedere dalla sola sequenza. Sul mutante, la loro variazione indica il meccanismo rotto. |
| 5 | La media dell'embedding su tutta la proteina non sente una singola sostituzione. | **Vista locale** di ±15 residui attorno alla variante, confrontata tra WT e mutante. |
| 6 | La mislocalizzazione patogenica dipende soprattutto da stabilità e inserzione in membrana (Lacoste et al., 2024). | **LLR di ESM** come proxy di stabilità e **ΔΔG di inserzione in membrana** (scala di Hessa) nei tratti TM. |
| 7 | Una variante patogenica in ClinVar non è necessariamente mislocalizzata. | ClinVar serve come supervisione debole. Le varianti con localizzazione misurata servono solo per valutare. |
| 8 | Bisogna misurare il guadagno senza contaminare il confronto. | Pool **`oro+fonti`**: stesse proteine di DeepLoc, più fonti di etichette. Il test HPA resta intatto. |
| 9 | Le probabilità devono essere affidabili. | **Soglie conformali** per classe, con un obiettivo di copertura del 90% verificato sul test. |

Altre scelte:
- **Split per omologia.** Si usano le 5 partizioni ufficiali di DeepLoc (≤30% di identità); le proteine aggiunte ereditano la partizione della proteina DeepLoc più simile.
- **Embedding.** ESM-2 con 5 riassunti per sequenza: media dell'ultimo layer, media del layer a 2/3, max, primi 40 e ultimi 40 residui.
- **Ensemble.** k-NN sugli embedding e soglie per classe, tutto scelto sulle predizioni out-of-fold.

## Struttura del repository

```
.
├── 01_dati.ipynb
├── 02_embedding_modello.ipynb
├── README.md
├── requirements.txt
├── .gitignore
├── cache/                    # download automatici (creata dal notebook 01, esclusa da git)
└── data/                     # tabelle intermedie, embedding e risultati (esclusa da git)
    └── esterni/
        ├── syngo_ontologies.xlsx   # (facoltativo) dal bulk download di SynGO
        ├── fonti/            # CSV scaricati e convertiti a mano (gene, classe)
        └── varianti/         # CSV di varianti con localizzazione misurata (gene, variante, mislocalizzata)
```

## Installazione

Serve Python ≥ 3.10.

```bash
git clone <url-del-repository>
cd <cartella>
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**Hardware:**
- Con GPU (≥ 12 GB) l'intero progetto con ESM-2 650M richiede poche ore.
- Su sola CPU, nel notebook 02 imposta `ESM = 'facebook/esm2_t30_150M_UR50D'` e `N_CONFIGS = 4`.

Su disco servono circa 1 GB per le cache e 1–2 GB per gli embedding.

## Guida rapida

1. **(Opzionale ma consigliato)** Scarica `syngo_ontologies.xlsx` da SynGO e mettilo in `data/esterni/` (vedi [Dati da scaricare a mano](#dati-da-scaricare-a-mano)). Le altre fonti manuali sono facoltative.
2. Esegui `01_dati.ipynb` dall'inizio alla fine. Il primo avvio scarica circa 600 MB; le esecuzioni successive usano `cache/`.
3. Esegui `02_embedding_modello.ipynb`. Il calcolo degli embedding si può interrompere e riprende da dove si era fermato.
4. Leggi i risultati in `data/` (vedi [Output](#output)).

> Se cambi l'elenco delle sequenze (per esempio `MAX_VARIANTS`, oppure aggiungi fonti che fanno entrare nuove proteine), cancella `data/emb_*` e `data/res_*` prima di rieseguire il notebook 02.

## Dati scaricati automaticamente

| Sorgente | Uso | Link |
|---|---|---|
| DeepLoc 2.0 – SwissProt (training, 5 partizioni di omologia) | Etichette sperimentali "oro" | [Swissprot_Train_Validation_dataset.csv](https://services.healthtech.dtu.dk/services/DeepLoc-2.0/data/Swissprot_Train_Validation_dataset.csv) |
| DeepLoc 2.0 – test HPA indipendente | Test finale, confronto con lo stato dell'arte | [hpa_testset.csv](https://services.healthtech.dtu.dk/services/DeepLoc-2.0/data/hpa_testset.csv) |
| UniProt umano (reviewed) | Sequenze, localizzazione con codici ECO, peptide segnale/TM/lipidazione, PTM, funzione, malattia, PDB | [rest.uniprot.org](https://rest.uniprot.org/) |
| UniProt, query "cervello" | Flag `is_brain` | idem |
| HPA – subcellular location | Etichette (Enhanced/Supported/Approved), multi-localizzazione | [subcellular_location.tsv.zip](https://www.proteinatlas.org/download/tsv/subcellular_location.tsv.zip) |
| HPA – RNA tissue consensus | Espressione nei tessuti cerebrali | [rna_tissue_consensus.tsv.zip](https://www.proteinatlas.org/download/tsv/rna_tissue_consensus.tsv.zip) |
| HPA – RNA single cell type | Espressione in neuroni e glia (testa neuronale) | [rna_single_cell_type.tsv.zip](https://www.proteinatlas.org/download/tsv/rna_single_cell_type.tsv.zip) |
| MitoCarta 3.0 | Inventario mitocondriale | [human.mitocarta3.0.html](https://www.broadinstitute.org/files/shared/metabolism/mitocarta/human.mitocarta3.0.html) |
| SynaptopathyDB | Target ausiliario "sinaptopatia" | [API genes](https://www.synaptopathydb.org/api/genes) |
| OpenCell | Localizzazione da tagging endogeno (gradi 3 e 2; il grado 1 resta "non osservato") | [opencell-localization-annotations.csv](https://opencell.sf.czbiohub.org/data/datasets/opencell-localization-annotations.csv) |
| ClinVar | Varianti missense (P/LP, B/LB, VUS) | [variant_summary.txt.gz](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/tab_delimited/variant_summary.txt.gz) |

## Dati da scaricare a mano

Sono tutti **facoltativi**: senza di loro il progetto gira. Servono a dare più dati alla testa neuronale e una verità sperimentale per valutare le varianti.

### 1. SynGO → presinapsi / postsinapsi (consigliato, 5 minuti)

Il notebook lo converte da solo.

1. Apri <https://www.syngoportal.org>. È un sito di ricerca: la ricerca non ti serve, cerca invece nel menu la voce **Data / Downloads**.
2. Clicca **"bulk download SynGO release …"**, prendendo la release più recente (per esempio `SynGO_bulk_download_release_20231201.zip`).
3. Scompatta lo zip e copia **`syngo_ontologies.xlsx`** in `data/esterni/`.

Il notebook 01 tiene solo i termini di componente cellulare (CC):
- quelli che contengono *presynap*, *synaptic vesicle* o *active zone* diventano `Presinapsi`;
- quelli che contengono *postsynap* diventano `Postsinapsi`.

### 2. OpenCell → già automatico

Non devi scaricare niente. Il notebook 01 prende da solo **Protein localization annotations (CSV)**:
<https://opencell.sf.czbiohub.org/data/datasets/opencell-localization-annotations.csv>

Gli altri file della pagina di download OpenCell (metadata della libreria, abbondanze proteiche, interazioni, dati grezzi) **non servono**.

### 3. Mappa spaziale di neuroni primari di topo — Itzhak et al., Cell Reports 2017

1. Apri l'articolo (open access): <https://doi.org/10.1016/j.celrep.2017.08.063>.
2. In fondo alla pagina, nella sezione **Supplemental Information**, scarica le tabelle Excel.
3. Cerca la tabella con **una riga per proteina** e l'**organello assegnato** nei neuroni. Ti servono una colonna con il simbolo genico e una con la predizione o classe di compartimento.
4. Mappa i compartimenti così:

   | Nome nella tabella (indicativo) | `classe` |
   |---|---|
   | Nucleus / Nuclear | `Nucleus` |
   | Cytosol | `Cytoplasm` |
   | Plasma membrane | `Cell membrane` |
   | ER | `Endoplasmic reticulum` |
   | Golgi | `Golgi apparatus` |
   | Mitochondria | `Mitochondrion` |
   | Lysosome | `Lysosome/Vacuole` |
   | Peroxisome | `Peroxisome` |

   Scarta gli altri: complessi grandi, endosomi, vescicole sinaptiche, proteine senza assegnazione.
5. Salva come `data/esterni/fonti/neuroni_itzhak.csv` (colonne `gene,classe`). I simboli di topo vanno bene così.

### 4. Assone e compartimento somatodendritico — Hobson et al., eLife 2022

1. Apri <https://elifesciences.org/articles/70921>.
2. Nella sezione **Additional files / Supplementary files** (ci sono anche i *source data* delle figure), scarica le tabelle.
3. Cerca gli elenchi di proteine **significativamente arricchite**:
   - negli **assoni dopaminergici striatali** → `Assone`;
   - nel **compartimento somatodendritico** (mesencefalo ventrale) → `Dendrite`. È un'approssimazione: questo compartimento include anche il soma.
4. Salva come `data/esterni/fonti/assone_hobson.csv`.

### 5. Varianti con localizzazione misurata — Lacoste et al., Cell 2024 (per la valutazione)

1. Apri <https://doi.org/10.1016/j.cell.2024.09.003>; la versione open access è su [PMC11568917](https://pmc.ncbi.nlm.nih.gov/articles/PMC11568917).
2. Nella sezione **Supplemental information**, scarica le tabelle Excel.
3. Cerca la tabella con **una riga per variante**: gene, cambio amminoacidico, esito dell'imaging (mislocalizzata oppure come il wild-type).
4. Crea `data/esterni/varianti/lacoste.csv` con:
   - `gene`: simbolo;
   - `variante`: formato breve, `p.Arg130Gly` → `R130G`;
   - `mislocalizzata`: 1 o 0.

> **Human Cell Map**: non è più usata. OpenCell e HPA coprono lo stesso ruolo (localizzazione sperimentale nelle cellule umane).

### Formato dei file in `data/esterni/fonti/`

```csv
gene,classe
TOMM20,Mitochondrion
SNAP25,Presinapsi
```

Valori ammessi per `classe`: `Cytoplasm`, `Nucleus`, `Extracellular`, `Cell membrane`, `Mitochondrion`, `Endoplasmic reticulum`, `Lysosome/Vacuole`, `Golgi apparatus`, `Peroxisome`, `Assone`, `Dendrite`, `Presinapsi`, `Postsinapsi`.

Due regole:
- Il nome del file diventa il nome della fonte, e il modello stima per ogni fonte quanto è affidabile.
- Una proteina presente nel file è considerata **assente** dai compartimenti che la fonte usa altrove ma non le assegna. Metti quindi solo assegnazioni di cui la fonte è sicura.

### Conversione con pandas (modello da adattare)

Cambia `COL_GENE`, `COL_LOC` e `MAPPA` secondo la tabella che hai scaricato:

```python
import pandas as pd
raw = pd.read_excel('tabella_supplementare.xlsx', sheet_name=0)
COL_GENE, COL_LOC = 'Gene names', 'Prediction'                    # nomi delle colonne nella tabella
MAPPA = {'Nucleus': 'Nucleus', 'Cytosol': 'Cytoplasm', 'Plasma membrane': 'Cell membrane',
         'ER': 'Endoplasmic reticulum', 'Golgi': 'Golgi apparatus', 'Mitochondria': 'Mitochondrion',
         'Lysosome': 'Lysosome/Vacuole', 'Peroxisome': 'Peroxisome'}
out = raw[[COL_GENE, COL_LOC]].rename(columns={COL_GENE: 'gene', COL_LOC: 'classe'})
out['gene'] = out['gene'].astype(str).str.split(';').str[0]       # se ci sono più nomi, tengo il primo
out['classe'] = out['classe'].map(MAPPA)
out.dropna().drop_duplicates().to_csv('data/esterni/fonti/neuroni_itzhak.csv', index=False)
```

## Cosa succede nei due notebook

### `01_dati.ipynb`

1. **Download in cache** di tutte le sorgenti automatiche, con retry. I file parziali hanno estensione `.part`, così un download interrotto non lascia file corrotti.
2. **DeepLoc**: training con partizioni di omologia e test HPA. Il test è collegato a UniProt per sequenza identica.
3. **Osservazioni da UniProt.** Ogni frase `Termine {ECO:...}` viene mappata a una classe e divisa in due fonti:
   - `uniprot_forte`: evidenza sperimentale o del curatore;
   - `uniprot_debole`: per similarità o predetta.

   La mappatura gestisce i casi difficili: "Cytoplasmic vesicle" non è citoplasma, "perinuclear region" non è nucleo, si usa solo l'isoforma canonica e le note libere vengono ignorate.
4. **Osservazioni da HPA** (`hpa_alta`, `hpa_approved`), **OpenCell**, **SynGO**, **MitoCarta** e fonti esterne.
5. **Regola delle assenze**, poi una tabella lunga `(proteina, fonte, classe, 0/1)`.
6. **Feature ereditate da FFFFF.** Sono calcolate direttamente da UniProt, HPA e SynaptopathyDB:
   - target ausiliari: peptide segnale, transit peptide, TM, ancore lipidiche, glicosilazione, NLS, multimero, PTM, aggregati, patologia cerebrale, sinaptopatia;
   - contesto genico: multi-localizzazione, shuttle nucleo-citoplasma, struttura PDB, ruolo nel trafficking.
7. **Flag `is_brain`** (query UniProt oppure nTPM cerebrale ≥ 10) e **`is_neuronal`** (nCPM neuronale ≥ 5).
8. **ClinVar.** Solo missense su GRCh38 in proteine cerebrali, classi nette (le "conflicting" sono escluse). Una variante si tiene solo se il residuo WT coincide con la sequenza UniProt.

### `02_embedding_modello.ipynb`

1. Sequenze da codificare: proteine e mutanti. Le proteine oltre 1022 aa sono troncate tenendo i due terminali.
2. Embedding ESM-2 su memmap ripartibili. Per le proteine con varianti salva anche gli embedding per-residuo e i log-prob.
3. Feature biologiche dalla sequenza: idrofobicità e carica N-terminale, tratti TM, NLS, KDEL, PTS1, CAAX, miristoilazione.
4. Tensore delle osservazioni `[proteina, fonte, classe]`, partizioni e controlli:
   - nessuna sequenza del test nel training;
   - vicinanza tra test e training;
   - copertura delle fonti.
5. **Grid search** con CV a 5 fold sulle partizioni di omologia. Quattro pool:

   | Pool | Proteine | Fonti | Canale |
   |---|---|---|---|
   | `oro` | DeepLoc | solo DeepLoc | — |
   | `oro+fonti` | DeepLoc | tutte | attivo |
   | `esteso` (`rumore=0`) | DeepLoc + umane aggiunte | tutte | fusione ingenua |
   | `esteso` (`rumore=1`) | DeepLoc + umane aggiunte | tutte | attivo |

6. Ablazioni e **tabella di affidabilità delle fonti** (sensibilità e specificità stimate).
7. Ensemble dei run migliori, k-NN, soglie MCC, soglie riscalate per il Jaccard e soglie conformali.
8. **Confronto con DeepLoc 2.0** sul test HPA.
9. Valutazione su **cervello** e **compartimenti neuronali**, e tabella delle probabilità per proteina.
10. **Varianti.** Calcola JSD tra WT e mutante, Δ dei compartimenti neuronali, Δ delle teste di meccanismo, spostamento locale, LLR, ΔΔG di inserzione e contesto. Produce due punteggi:
    - non supervisionato (`score_misloc`);
    - supervisionato su ClinVar, con CV raggruppata per gene (`score_supervisionato`).

    Riporta anche l'AUROC su ClinVar e sulle varianti misurate.

## Output

| File | Contenuto |
|---|---|
| `data/proteine.parquet` | Proteine, sequenze, flag cerebrali/neuronali, target ausiliari, contesto |
| `data/osservazioni.parquet` | Osservazioni `(pid, fonte, classe, obs)` |
| `data/clinvar_missense.parquet` | Varianti ClinVar filtrate |
| `data/confronto_stato_arte.xlsx` | Metriche sul test HPA accanto a DeepLoc 2.0 |
| `data/probabilita_proteine_cervello.xlsx` | Per ogni proteina cerebrale: probabilità per compartimento, profilo %, compartimenti predetti, insieme conformale al 90% |
| `data/varianti_mislocalizzazione.xlsx` | Per ogni variante: tutti i segnali, meccanismo principale, i due punteggi |

## Valutazione e confronto con lo stato dell'arte

Il protocollo è identico a quello di DeepLoc 2.0:
- training sulle 5 partizioni SwissProt;
- soglie per classe scelte sul training;
- metriche sulle 6 classi HPA.

Il test HPA non viene mai usato per scegliere iperparametri, soglie o pesi.

Valori pubblicati da battere (Thumuluri et al., NAR 2022, Tabella 2):

| Modello | Accuracy | Jaccard | Micro F1 | Macro F1 |
|---|---|---|---|---|
| DeepLoc 2.0 (ESM1b) | 0.34 | 0.48 | 0.57 | 0.44 |
| DeepLoc 2.0 (ProtT5) | 0.39 | 0.53 | 0.60 | 0.46 |
| Light Attention (ProtT5) | 0.45 | 0.52 | 0.56 | 0.43 |

Il notebook riporta due punti operativi:
- **soglie MCC**, come DeepLoc;
- **soglie Jaccard**, che riportano il numero medio di etichette per proteina vicino a quello reale.

## Test interni

Il codice è stato eseguito end-to-end su dati finti ma con i formati reali delle sorgenti, costruiti con casi difficili e scenari pianificati. Al posto di ESM-2 c'era uno stub, quindi **questi numeri verificano la metodologia, non le prestazioni finali**.

| Verifica | Esito |
|---|---|
| Parsing UniProt (perinucleare, vescicole, isoforme, note, evidenze ECO), alias HPA, duplicati | corretto |
| ClinVar: esclusione di conflicting, GRCh37, nonsense, residuo WT errato | corretto |
| Canale di rumore contro fusione ingenua (macro-AUPRC CV sulle etichette DeepLoc) | 0.508 contro 0.484 (protocollo DeepLoc: 0.495) |
| Proteine note solo tramite fonti rumorose, contro la verità nascosta | 0.517 contro 0.496 |
| Il canale distingue una fonte buona da una rumorosa | sì: sens/spec 0.87/0.96 contro 0.74/0.77 |
| Target ausiliari multi-task | 0.499 contro 0.492 senza |
| Varianti pianificate (rottura del peptide segnale o dell'elica TM) | AUROC 0.96–0.99 (JSD, Δ meccanismo); il meccanismo principale è identificato correttamente |
| Copertura conformale sul test | 0.88 con obiettivo 0.90 |

## Limiti noti

- **Spostamento di distribuzione e conformale.** Il test HPA è ridotto per omologia rispetto al training, quindi la copertura conformale scende leggermente sotto l'obiettivo. Per una garanzia più stretta abbassa `EPS`.
- **Probabilità latenti nei pool multi-fonte.** Possono risultare poco calibrate. La scala delle soglie tende al bordo della griglia: controlla la stampa della sezione 7.
- **Golgi e lisosoma.** Restano le classi più difficili, come per tutti i metodi pubblicati.
- **Fonti di topo.** Le mappe neuronali di topo usano l'ortologia per simbolo genico: è un'approssimazione, e il canale di rumore ne stima l'affidabilità.
- **Isoforme.** Si usa solo l'isoforma canonica UniProt. Le isoforme specifiche del cervello (microesoni) non sono modellate.
- **Proxy di stabilità.** L'LLR di ESM è un proxy della stabilità, non una misura di ΔΔG di folding.

## Riferimenti

- Thumuluri V. et al. *DeepLoc 2.0: multi-label subcellular localization prediction using protein language models.* Nucleic Acids Research, 2022. [Dati](https://services.healthtech.dtu.dk/services/DeepLoc-2.0/4-Data.php)
- Stärk H. et al. *Light attention predicts protein location from the language of life.* Bioinformatics Advances, 2021. [Codice](https://github.com/HannesStark/protein-localization)
- Lin Z. et al. *Evolutionary-scale prediction of atomic-level protein structure with a language model* (ESM-2). Science, 2023.
- Lacoste J. et al. *Pervasive mislocalization of pathogenic coding variants underlying human disorders.* Cell, 2024. [doi](https://doi.org/10.1016/j.cell.2024.09.003)
- Brandes N. et al. *Genome-wide prediction of disease variant effects with a deep protein language model.* Nature Genetics, 2023.
- Hessa T. et al. *Recognition of transmembrane helices by the endoplasmic reticulum translocon.* Nature, 2005.
- Itzhak D. N. et al. *A mass spectrometry-based approach for mapping protein subcellular localization reveals the spatial proteome of mouse primary neurons.* Cell Reports, 2017.
- Hobson B. D. et al. *Subcellular proteomics of dopamine neurons in the mouse brain.* eLife, 2022.
- Koopmans F. et al. *SynGO: an evidence-based, expert-curated knowledge base for the synapse.* Neuron, 2019.
- Cho N. H. et al. *OpenCell: endogenous tagging for the cartography of human cellular organization.* Science, 2022.
- Rath S. et al. *MitoCarta3.0.* Nucleic Acids Research, 2021.
- Dawid A. P., Skene A. M. *Maximum likelihood estimation of observer error-rates using the EM algorithm.* JRSS C, 1979.

---

**Autore:** `<Nome Cognome>` — Università degli Studi di Salerno
**Licenza:** `<da scegliere, es. MIT>`
