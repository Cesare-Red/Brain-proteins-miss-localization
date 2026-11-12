# NeuroLoc-Δ

Probabilità di localizzazione subcellulare delle proteine del cervello umano (sangue escluso) in 14 classi: le 9 di
DeepLoc 2.0 più assone, dendrite, presinapsi, postsinapsi e mielina. Con varianti missense e malattie neurologiche.

## Struttura

```
neuroloc-delta/
├── 01_dati.ipynb                        dati, etichette, perimetro cerebrale, varianti
├── 02_embedding_modello.ipynb           embedding, omologia, rete, confronto con lo stato dell'arte
├── 03_cervello_varianti_malattie.ipynb  calibrazione, percentuali, varianti, malattie, pagina HTML
├── requirements.txt
├── docs/
│   ├── NeuroLoc_articolo_revisione.docx
│   └── riferimenti_corretti.md          correzioni alla bibliografia dalla revisione
└── data/
    └── esterni/                         file messi a mano (già inclusi, tranne DeepLoc 2.0)
        ├── human.mitocarta3.0.xls       MitoCarta 3.0 (Broad Institute), foglio B = 19.247 geni
        ├── syngo/                       SynGO bulk download: annotations, genes, ontologies (.xlsx)
        ├── wefers/                      test HOU e risultati pubblicati di Wefers et al. 2026 (lo 03 li riscarica se mancano)
        ├── fonti/                       fonti in più, facoltative (vedi LEGGIMI.md)
        ├── varianti/                    varianti misurate in più, facoltative (vedi LEGGIMI.md)
        └── nuove/                       FASTA di proteine nuove da predire con lo 03, facoltativo
```

Tutto il resto lo scaricano o lo generano i notebook: `cache/` (download dello 01) e `data/` (tabelle, embedding,
strutture AlphaFold, modelli, figure). Sono in `.gitignore`: non vanno su GitHub.

## Installazione

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Serve una GPU per lo 02 (provato su RTX A5000, 24 GB). I modelli ESM-2, ProtT5 e PubMedBERT si scaricano da Hugging Face
al primo avvio.

## Esecuzione

I notebook si lanciano **dalla cartella principale**, in ordine: tutti i percorsi sono relativi (`data/`, `cache/`).

1. **01_dati** — scarica DeepLoc 2.0, UniProt, HPA, GOA, OpenCell, LOPIT, BioID, ClinVar, Lacoste et al. e le
   strutture AlphaFold (la prima volta richiede alcune ore, poi resta in `data/alphafold/`). Legge MitoCarta e SynGO da
   `data/esterni/`. Se mancano, lo dice con un avviso e prosegue senza.
2. **02_embedding_modello** — embedding, partizioni per omologia (allineamenti locali, circa 10–20 minuti), rete,
   ablazioni, confronto con la ricetta DeepLoc 2.0. Scrive `data/esiti_02.npz` per lo 03.
3. **Passo manuale (DeepLoc 2.0 ufficiale)** — lo 02 scrive `data/hpa_test_per_deeploc2_*.fasta`. Caricali sul server
   DeepLoc 2.0 (DTU Health Tech), scarica i CSV dei risultati e salvali come `data/esterni/deeploc2_1.csv`,
   `deeploc2_2.csv`, … Poi rilancia lo 02 dalla sezione del confronto. Senza questi file il confronto usa solo la
   ricetta DeepLoc 2.0 riprodotta. Se il test cambia (nuovo 01), i FASTA cambiano e il passo va rifatto.
4. **03_cervello_varianti_malattie** — calibrazione, percentuali, tabella finale, varianti, malattie, casi di studio,
   pagina `data/neuroloc_delta.html` (si apre senza server) e contesto di Wefers et al.

Se cambia l'elenco delle proteine (nuovo 01), cancella `data/emb_*` e `data/res_*` prima di rilanciare lo 02.

## Uscite principali

| File | Contenuto |
|------|-----------|
| `data/scelta_metodologia.xlsx` | AUC/AUPRC di tutti i modelli, IC bootstrap, differenze dallo stato dell'arte |
| `data/omologia_allineamenti.parquet` | coppie allineate, identità e copertura |
| `data/probabilita_proteine_cervello.xlsx` | probabilità e percentuali per proteina |
| `data/varianti_mislocalizzazione.xlsx` | punteggio di mislocalizzazione delle varianti |
| `data/confronto_wefers.xlsx`, `data/roc_wefers.png` | contesto sul test HOU (non appaiato) |
| `data/neuroloc_delta.html` | pagina consultabile di tutte le proteine |

## Fonti dei file in `data/esterni/`

- MitoCarta 3.0: Rath S et al., *Nucleic Acids Res* 2021;49:D1541–D1547 (broadinstitute.org/mitocarta).
- SynGO: Koopmans F et al., *Neuron* 2019;103:217–234 (syngoportal.org, bulk download).
- Test HOU: Wefers et al. 2026, repository `CellProfiling/seq2loc_benchmark` su GitHub.
