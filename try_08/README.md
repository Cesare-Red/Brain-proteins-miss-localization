# NeuroLoc-Δ

Probabilità di localizzazione subcellulare delle proteine umane espresse nel cervello (sangue escluso): le 9 classi di
DeepLoc 2.0 più assone, dendrite, presinapsi, postsinapsi e mielina. Modello principale di **sola sequenza**.

Stato: applicate la revisione del 14 novembre, la verifica del 16 novembre e la revisione critica del 18 novembre 2026
(`docs/DOMANDE_ESPERTO.md`: domande da esperto, risposte e correzioni). In `docs/` anche le due revisioni
(`STATO_CODICE.md`, `VERIFICA_METODOLOGIA.md`) e le risposte punto per punto (`RISPOSTA_REVISIONE.md`,
`RISPOSTA_VERIFICA.md`). 02 e 03 sono stati eseguiti dall'inizio alla fine su dati sintetici (test del codice). **I notebook in questa cartella sono senza output: i numeri dell'articolo vanno presi dalla
prossima esecuzione completa (Restart & Run All di 01, 02, 03), non da quelle precedenti.**

## Struttura

```
neuroloc-delta/
├── 01_dati.ipynb                        dati, etichette, perimetro, test HPA, varianti
├── 02_embedding_modello.ipynb           embedding, omologia, rete, scelta del modello, confronto
├── 03_cervello_varianti_malattie.ipynb  calibrazione, fusione, percentuali, varianti, malattie, pagina HTML
├── requirements.txt
├── docs/
│   ├── STATO_CODICE.md, VERIFICA_METODOLOGIA.md       le due revisioni
│   ├── RISPOSTA_REVISIONE.md, RISPOSTA_VERIFICA.md    le risposte punto per punto
│   ├── NeuroLoc_articolo_revisione.docx bozza (da riallineare ai nuovi risultati)
│   └── riferimenti_corretti.md
└── data/esterni/                        file messi a mano
    ├── human.mitocarta3.0.xls           MitoCarta 3.0, foglio B
    ├── syngo/                           annotations, genes, ontologies (.xlsx)
    ├── wefers/                          test HOU e risultati pubblicati (lo 03 li riscarica se mancano)
    ├── deeploc2_*.csv                   facoltative: predizioni del server di DeepLoc 2.0 (non servono)
    ├── fonti/, varianti/, nuove/        facoltative (vedi i LEGGIMI.md)
```

`cache/` (download) e il resto di `data/` (tabelle, embedding, strutture, modelli, figure) li creano i notebook e
sono in `.gitignore`.

## Esecuzione di riferimento

Dalla cartella principale, ognuno con **Kernel → Restart & Run All**, in ordine:

1. **01_dati** — scarica i dati (anche la tabella ID di UniProt per collegare il test HPA) e scrive
   `data/versioni_dati.csv` con data e impronta SHA-256 di ogni file: va messo nel repository insieme ai risultati.
   Controlla la riga "Test HPA di DeepLoc 2.0: … collegate a UniProt": le non collegate devono essere poche.
2. **02_embedding_modello** — la cache degli embedding è legata a un'impronta delle sequenze: dopo un nuovo 01 la
   cache vecchia viene spostata in `data/cache_vecchia/` e tutto si ricalcola (circa 1 ora di embedding su RTX A5000,
   30–35 minuti di allineamenti, circa 1 ora di rete). La sezione 8d rifà **DeepLoc 2.0** e **Wefers ProtT5-MHA** sulla
   tua macchina (vedi sotto): in più circa 30 minuti di embedding per residuo (**circa 18 GB su disco** in
   `data/res_prott5_4k.npy`), 30–40 minuti di DeepLoc 2.0, circa 1–2 ore di Wefers (sequenze intere fino a 4.000
   residui), pochi minuti per le due reti NeuroLoc-Δ con lo stesso protocollo. La ricerca degli omologhi del test è più
   profonda: circa 15–25 minuti in più di allineamenti. Le stime non sono misurate sulla tua macchina.
3. **03_cervello_varianti_malattie**.

## Confronto con lo stato dell'arte, a parità di condizioni

Tutto in locale, nello 02 (sezione 8d). Un solo protocollo di addestramento per tutti, quello di **DeepLoc 2.0**,
preso dal suo codice:
- etichette curate di DeepLoc (proteine umane, 9 classi + membrana);
- training sulle 4 partizioni diverse da k e validazione sul 9% scelto a caso, la stessa divisione per tutti;
- focal loss con i pesi di DeepLoc, AdamW lr 1e-3, ReduceLROnPlateau, arresto dopo 5 epoche, massimo 14;
- 3 semi × 5 fold.

**Nessun iperparametro è scelto guardando le nostre proteine.** Quattro righe:
- **DeepLoc 2.0 rifatto** (`TviNet/DeepLoc-2.0`): la sua architettura;
- **Wefers ProtT5-MHA** (`CellProfiling/seq2loc_benchmark`): la sua architettura. Le scelte che nel loro lavoro venivano
  dalla griglia prendono il valore di DeepLoc 2.0: 4.000 residui, nessun dropout, focal loss, lotti per residui;
- **NeuroLoc-Δ, sola ProtT5**: gli stessi ingressi delle due repliche (embedding per residuo di ProtT5), riassunti in 7
  vettori fissi. Misura solo il metodo;
- **NeuroLoc-Δ, rappresentazione completa**: i nostri ingressi (ESM-2 + ProtT5, descrittori, motivi, struttura), fissati
  a priori.

Il sistema completo (dati in più, configurazione scelta in CV) resta come contesto.

**Risultato principale:**
- **test HPA** (02, sezione 10; fogli "stato dell'arte" di `scelta_metodologia.xlsx`);
- **HOU fuori da DeepLoc**: le proteine HOU di cui nessun metodo ha mai visto un'etichetta (03, sezione 12,
  `confronto_wefers.xlsx`).

**Confronto con i valori pubblicati** (02, sezione 11d; 03, sezione 13): il nostro modello con la migliore AUPRC in CV
(sola sequenza) contro i numeri stampati negli articoli:
- **test HPA**: Tabella 2 di DeepLoc 2.0 (YLoc+, DeepLoc 1.0, Fuel-mLoc, LAProtT5, DeepLoc 2.0), stesse metriche e stesse
  regole di soglia, ogni gene pesato quante voci aveva nel benchmark;
- **test HOU**: AUC, MCC, richiamo e precisione per classe pubblicati da Wefers et al.;
- curve ROC finali con i metodi pubblicati nel piano ROC (punti di lavoro; per DeepLoc, dove c'è solo l'MCC, la curva
  dei punti con lo stesso MCC).

Le AUC di DeepLoc 2.0 sul test HPA sono solo nella Tabella S7 del supplemento: se le aggiungi in
`data/esterni/deeploc2_auc_hpa.csv`, entrano nel confronto. Come migliorare rispetto ai metodi pubblicati:
`docs/MIGLIORARE_PRESTAZIONI.md`.

Le differenze sono appaiate (bootstrap) con il p corretto con Holm. Limite dichiarato: con circa 6.400 proteine umane i
modelli ad attenzione possono essere svantaggiati. La conclusione vale a questa scala di dati.

Poi commit dei notebook eseguiti, di `data/versioni_dati.csv` e di `requirements.lock.txt` (lo scrive lo 01).

Facoltativo: conferma dell'omologia con MMseqs2 (vedi `docs/RISPOSTA_VERIFICA.md`, punto B3).

## Cosa si può affermare (e cosa no)

- Il modello principale è di sola sequenza, scelto come il più semplice tra gli equivalenti in CV; la scheda testuale è
  una variante dichiarata "aumentata dalla conoscenza".
- I contributi (modello, dati, fonti HPA, attenzione sui residui, scheda) sono quelli stampati nella sezione 10 dello
  02 con il loro intervallo: un contributo il cui intervallo contiene lo zero non è dimostrato.
- La tabella finale tiene separate predizione e annotazioni; per i compartimenti neuronali dà un percentile.
- Il confronto con lo stato dell'arte è quello **a parità di condizioni** (stesse etichette, divisione e protocollo di
  DeepLoc 2.0, nessun iperparametro scelto sui nostri dati), letto sul test HPA e sulle HOU fuori da DeepLoc. Va scritto "DeepLoc 2.0 riaddestrato sui nostri dati", non
  "DeepLoc 2.0": il modello pubblicato è addestrato su 28.000 proteine eucariote. Il sistema completo usa dati in più ed
  è un confronto diverso; sul test HOU non è indipendente (le etichette HOU vengono dalle stesse banche dati).
- Varianti: `score_variante` è una priorità per patogenicità. I segnali di spostamento della localizzazione non
  predicono la mislocalizzazione misurata: risultato negativo, riportato con gli intervalli di confidenza.
- Compartimenti neuronali: positivi annotati contro non annotati, senza test esterno; la mielina non è valutabile.
- "Cervello": proteine espresse nel cervello (circa l'80% di Swiss-Prot umano), etichette da linee cellulari.

## Uscite principali

| File | Contenuto |
|------|-----------|
| `data/versioni_dati.csv` | inventario dei dati usati (data, dimensione, SHA-256) |
| `data/scelta_metodologia.xlsx` | tutti i modelli, IC bootstrap, strati di omologia del test |
| `data/omologia_allineamenti.parquet` | coppie allineate, identità e copertura |
| `data/verifica_fusione.xlsx` | calibrazione e saturazione dopo la fusione con le misure |
| `data/tabella_annotate_e_predette.xlsx` | quante annotazioni la predizione ritrova, quante candidate sono nuove |
| `data/varianti_valutazione.xlsx` | AUROC delle varianti con IC, anche entro proteina |
| `data/probabilita_proteine_cervello.xlsx` | probabilità e percentuali per proteina |
| `data/confronto_wefers.xlsx` | test HOU: NeuroLoc-Δ, DeepLoc 2.0 e Wefers rifatti sulle stesse proteine, differenze appaiate |
| `data/confronto_pubblicati.xlsx`, `data/roc_confronto_pubblicati.png` | test HPA: il nostro modello migliore contro i valori pubblicati da DeepLoc 2.0 (Tabella 2) |
| `data/confronto_pubblicati_hou.xlsx`, `data/roc_confronto_pubblicati_hou.png` | test HOU: il nostro modello migliore contro i valori pubblicati da Wefers et al. |
| `data/neuroloc_delta.html` | pagina consultabile |

## Fonti dei file in `data/esterni/`

- MitoCarta 3.0: Rath S et al., *Nucleic Acids Res* 2021;49:D1541–D1547.
- SynGO: Koopmans F et al., *Neuron* 2019;103:217–234 (bulk download).
- Test HOU: Wefers et al. 2026, repository `CellProfiling/seq2loc_benchmark`.
- DeepLoc 2.0: Thumuluri V et al., *Nucleic Acids Res* 2022;50:W228–W234; codice `TviNet/DeepLoc-2.0`; i segnali di
  smistamento per residuo (`SortingSignalsSwissprot.csv`) li scarica lo 01 dal sito DTU.
