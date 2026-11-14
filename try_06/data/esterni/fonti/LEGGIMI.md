# Fonti di localizzazione aggiuntive (facoltative)

Lo 01 legge ogni file `*.csv` di questa cartella come una fonte in più per il canale di rumore; il nome del file diventa
il nome della fonte. Formato: una riga per coppia gene–compartimento.

```
gene,classe
SYN1,Presinapsi
DLG4,Postsinapsi
```

Classi ammesse: le 9 di DeepLoc (`Cytoplasm`, `Nucleus`, `Extracellular`, `Cell membrane`, `Mitochondrion`,
`Endoplasmic reticulum`, `Lysosome/Vacuole`, `Golgi apparatus`, `Peroxisome`) e `Assone`, `Dendrite`, `Presinapsi`,
`Postsinapsi`, `Mielina`. Vuota va bene: OpenCell, BioID, LOPIT, MitoCarta e SynGO entrano già da soli.
