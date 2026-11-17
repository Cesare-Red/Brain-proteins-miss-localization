# NeuroLoc-Δ

Probabilità di localizzazione subcellulare delle proteine umane espresse nel cervello (sangue escluso): le 9 classi di
DeepLoc 2.0 più assone, dendrite, presinapsi, postsinapsi e mielina. Modello principale di **sola sequenza**.

Stato: applicate la revisione del 14 novembre e la verifica del 16 novembre 2026. In `docs/`: le due revisioni
(`STATO_CODICE.md`, `VERIFICA_METODOLOGIA.md`) e le risposte punto per punto (`RISPOSTA_REVISIONE.md`,
`RISPOSTA_VERIFICA.md`). **I notebook in questa cartella sono senza output: i numeri dell'articolo vanno presi dalla
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
   `data/res_prott5_4k.npy`), 30–40 minuti di DeepLoc 2.0, 1,5–2 ore di Wefers. È ripartibile: se si interrompe,
   rilancia la cella.
3. **03_cervello_varianti_malattie**.

## Confronto con lo stato dell'arte, tutto in locale

DeepLoc 2.0 "Accurate" (repository `TviNet/DeepLoc-2.0`) e il ProtT5-MHA di Wefers et al. 2026 (repository
`CellProfiling/seq2loc_benchmark`) sono ricopiati dal loro codice pubblico (architettura, perdite, ottimizzatore,
arresto, lotti; per Wefers anche la loro griglia di 18 configurazioni) e addestrati **sugli stessi dati** della nostra
rete: le etichette curate di DeepLoc delle proteine umane, le stesse partizioni di omologia, lo stesso test HPA, 3 semi ×
5 fold per tutti. Contro di loro si mette la nostra metodologia sugli stessi dati (stadio `abl_solo_deeploc`), poi il
nostro sistema completo con i dati in più. Confronto appaiato, con intervalli bootstrap delle differenze:
- **test HPA** (02, sezione 10, foglio "stato dell'arte" di `scelta_metodologia.xlsx`);
- **test HOU di Wefers**, fuori fold, sulle stesse proteine per tutti i metodi (03, sezione 12, `confronto_wefers.xlsx`),
  anche sulle sole proteine HOU che non sono tra le curate di DeepLoc.
Le differenze dagli originali (dati umani invece che eucarioti, 9 classi senza plastidio, 3 semi, validazione di Wefers
su una partizione diversa da quella di valutazione, griglia di Wefers sul fold 0) sono elencate nella sezione 8d.
Il server di DeepLoc non serve; se un giorno lo userai, la sezione 9 aggiunge anche le sue predizioni ufficiali.

Poi commit dei notebook eseguiti, di `data/versioni_dati.csv` e di `requirements.lock.txt` (lo scrive lo 01).

Facoltativo: conferma dell'omologia con MMseqs2 (vedi `docs/RISPOSTA_VERIFICA.md`, punto B3).

## Cosa si può affermare (e cosa no)

- Il modello principale è di sola sequenza, scelto come il più semplice tra gli equivalenti in CV; la scheda testuale è
  una variante dichiarata "aumentata dalla conoscenza".
- I contributi (modello, dati, fonti HPA, attenzione sui residui, scheda) sono quelli stampati nella sezione 10 dello
  02 con il loro intervallo: un contributo il cui intervallo contiene lo zero non è dimostrato.
- La tabella finale tiene separate predizione e annotazioni; per i compartimenti neuronali dà un percentile.
- Il confronto con lo stato dell'arte è quello **a parità di dati**: DeepLoc 2.0 e Wefers ProtT5-MHA rifatti dal loro
  codice, contro la nostra metodologia sugli stessi dati. Va scritto "DeepLoc 2.0 riaddestrato sui nostri dati", non
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
| `data/neuroloc_delta.html` | pagina consultabile |

## Fonti dei file in `data/esterni/`

- MitoCarta 3.0: Rath S et al., *Nucleic Acids Res* 2021;49:D1541–D1547.
- SynGO: Koopmans F et al., *Neuron* 2019;103:217–234 (bulk download).
- Test HOU: Wefers et al. 2026, repository `CellProfiling/seq2loc_benchmark`.
- DeepLoc 2.0: Thumuluri V et al., *Nucleic Acids Res* 2022;50:W228–W234; codice `TviNet/DeepLoc-2.0`; i segnali di
  smistamento per residuo (`SortingSignalsSwissprot.csv`) li scarica lo 01 dal sito DTU.
