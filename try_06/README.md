# NeuroLoc-Δ

Probabilità di localizzazione subcellulare delle proteine umane espresse nel cervello (sangue escluso): le 9 classi di
DeepLoc 2.0 più assone, dendrite, presinapsi, postsinapsi e mielina. Modello principale di **sola sequenza**.

Stato: revisione del 14 novembre 2026 applicata (`docs/STATO_CODICE.md` = la revisione, `docs/RISPOSTA_REVISIONE.md` =
cosa è stato cambiato punto per punto). **I notebook sono senza output: i numeri dell'articolo vanno presi dalla
prossima esecuzione completa**, non da quelle precedenti.

## Struttura

```
neuroloc-delta/
├── 01_dati.ipynb                        dati, etichette, perimetro, test HPA, varianti
├── 02_embedding_modello.ipynb           embedding, omologia, rete, scelta del modello, confronto
├── 03_cervello_varianti_malattie.ipynb  calibrazione, fusione, percentuali, varianti, malattie, pagina HTML
├── requirements.txt
├── docs/
│   ├── STATO_CODICE.md                  la revisione
│   ├── RISPOSTA_REVISIONE.md            la risposta punto per punto
│   ├── NeuroLoc_articolo_revisione.docx bozza (da riallineare ai nuovi risultati)
│   └── riferimenti_corretti.md
└── data/esterni/                        file messi a mano
    ├── human.mitocarta3.0.xls           MitoCarta 3.0, foglio B
    ├── syngo/                           annotations, genes, ontologies (.xlsx)
    ├── wefers/                          test HOU e risultati pubblicati (lo 03 li riscarica se mancano)
    ├── deeploc2_*.csv                   predizioni ufficiali di DeepLoc 2.0: DA AGGIUNGERE (vedi sotto)
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
   30–35 minuti di allineamenti, circa 1 ora di rete).
3. **DeepLoc 2.0 ufficiale (passo manuale, necessario per il confronto con lo stato dell'arte)** — lo 02 scrive
   `data/hpa_test_per_deeploc2_*.fasta`. Caricali su https://services.healthtech.dtu.dk/services/DeepLoc-2.0/
   (modello "High-quality (Accurate)"), salva i CSV come `data/esterni/deeploc2_1.csv`, `deeploc2_2.csv`, … e rilancia
   lo 02 dalla sezione 9. Senza questi file lo 02 dice "Confronto con DeepLoc 2.0: NON disponibile": il "riferimento
   semplice" non è DeepLoc 2.0 e non va presentato come stato dell'arte.
4. **03_cervello_varianti_malattie**.

Poi `pip freeze > requirements.lock.txt` e commit di notebook eseguiti, `data/versioni_dati.csv` e lock.

## Cosa si può affermare (e cosa no)

- Il modello principale è di sola sequenza; la scheda testuale è una variante dichiarata "aumentata dalla conoscenza".
- Il confronto con lo stato dell'arte è quello appaiato con DeepLoc 2.0 ufficiale, se i CSV ci sono; il contesto di
  Wefers et al. non è appaiato ("dentro l'intervallo" = parità).
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
| `data/varianti_valutazione.xlsx` | AUROC delle varianti con IC, anche entro proteina |
| `data/probabilita_proteine_cervello.xlsx` | probabilità e percentuali per proteina |
| `data/confronto_wefers.xlsx` | contesto sul test HOU (non appaiato) |
| `data/neuroloc_delta.html` | pagina consultabile |

## Fonti dei file in `data/esterni/`

- MitoCarta 3.0: Rath S et al., *Nucleic Acids Res* 2021;49:D1541–D1547.
- SynGO: Koopmans F et al., *Neuron* 2019;103:217–234 (bulk download).
- Test HOU: Wefers et al. 2026, repository `CellProfiling/seq2loc_benchmark`.
