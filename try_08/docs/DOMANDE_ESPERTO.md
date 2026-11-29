# NeuroLoc-Δ · Domande critiche sul progetto (revisioni del 18 e 19 novembre 2026)

Domande che un revisore esperto farebbe, con la risposta verificata sul codice. Dove la risposta ha richiesto una
modifica, è indicato **→ corretto**.

## Esecuzione e ambiente

**I notebook partono da zero senza errori?** No, nella versione del 18 novembre. Togliendo i commenti dalla cella 0
dello 02 è sparita anche la riga `import os, sys, time, math, platform`, e la cella usa `os.environ` e
`platform.processor()`: si fermava con `NameError`. **→ corretto** (rimessa la riga, senza i commenti).

**Il KeyboardInterrupt nella cella dell'embedding per residuo è un errore?** No: è un'interruzione a mano durante
`import transformers`. Su Windows il primo import di transformers (e di jinja2) può richiedere minuti; non va
interrotto.

**Su Windows i file si leggono e scrivono bene?** Quasi tutto passa da pandas (UTF-8 di default) o da scritture con
`encoding='utf-8'`. Restava la lettura di `go-basic.obo` con la codifica predefinita di Windows (cp1252), che
rovina i nomi dei termini con lettere non ASCII. **→ corretto**.

**Lo spazio su disco per l'embedding per residuo è controllato?** Prima veniva solo stampato. Ora la cella si ferma
con un messaggio chiaro se mancano i circa 18 GB, invece di fallire a metà scrittura. **→ corretto**.

**Il test end-to-end conferma che tutto gira?** Sì. Lo 02 e lo 03 sono stati eseguiti dall'inizio alla fine su dati
sintetici con lo schema vero dello 01: modelli di linguaggio finti, nessuna rete esterna, epoche ridotte. Il test
dimostra che il codice gira; i numeri veri escono solo dall'esecuzione completa sulla tua macchina.

## Dati e valutazione

**Ogni proteina ha una predizione fuori fold da ogni modello?** No, prima. La rete e le due repliche (DeepLoc 2.0,
Wefers) predicevano tutta la partizione k; logistica, k-NN, alberi, ensemble annidato, NeuroLoc-LA e partner
molecolari predicevano solo le proteine curate. Se la CV sceglieva uno di questi modelli:
- nello 03 le proteine aggiunte prendevano il logit della rete `cervello` invece di quello del modello scelto;
- nel test HOU il sottoinsieme più informativo ("HOU fuori da DeepLoc") restava vuoto.

**→ corretto.** Ora ogni modello predice tutta la partizione k: curate e aggiunte. Nessuna perdita di informazione:
le aggiunte stanno nella partizione delle loro omologhe curate, e quelle con un'omologa curata in un altro fold restano
fuori dalle valutazioni (`valutabile`). Un `assert` nello 02 lo controlla per tutti i modelli.

**La logistica sceglie C sul fold 0 e poi riporta la CV su tutti i fold: è ottimistica?** Leggermente, sul solo fold 0
e su 3 valori di C. È un riferimento, non il modello scelto. Si lascia così ed è dichiarato.

**Il confronto con lo stato dell'arte è alla pari?** Prima no, per quattro motivi:
- la nostra rete era scelta tra 48 + 5 configurazioni guardando le proteine curate su cui poi veniva valutata;
- i metodi avevano impegni di regolazione diversi: nessuno per DeepLoc, 18 combinazioni per Wefers, oltre 50 per noi;
- i metodi avevano training diversi: la nostra rete e Wefers usavano 3 partizioni, DeepLoc 4;
- ingressi diversi, presentati come "cambia solo la metodologia".

La riga "solo etichette DeepLoc" usava in più le frazioni misurate e le perdite ausiliarie. **→ corretto**:
- un solo protocollo per tutti, quello di DeepLoc 2.0 (02, sezione 8d), con la stessa divisione per la validazione e
  nessun iperparametro scelto sui nostri dati;
- Wefers prende i valori di DeepLoc dove il loro lavoro usava la griglia;
- NeuroLoc-Δ entra con due righe: sola ProtT5 (gli stessi ingressi delle repliche) e rappresentazione completa;
- lo stadio "Dati: …" è rinominato per quello che fa.

**Qual è il risultato principale?** Il test HPA e il gruppo "HOU fuori da DeepLoc": etichette indipendenti, proteine
che nessuna scelta ha guardato. Per le righe a parità di condizioni anche la CV sulle curate è pulita. La CV del
sistema completo resta ottimistica (configurazione scelta su di essa).

**Le differenze sono corrette per i confronti multipli?** Ora sì: p dal bootstrap appaiato e correzione di Holm dentro
ogni famiglia di confronti, nello 02 e nello 03. **→ corretto**.

**Il test HOU è indipendente?** Per i tre metodi sugli stessi dati quasi sì, sulle proteine HOU fuori da DeepLoc: non
hanno mai visto un'etichetta di quelle proteine. Per il sistema completo no: le etichette HOU vengono da HPA, OpenCell
e UniProt, che sono anche fonti di training.

**I valori dei metodi pubblicati sono quelli stampati negli articoli?** Ora sì, in una sezione a parte (02, 11d;
03, 13):
- DeepLoc 2.0, Tabella 2, test HPA: tutti i sei metodi della tabella, con le stesse metriche e le stesse regole di
  soglia; ogni gene pesa quante voci aveva nel benchmark;
- Wefers et al., test HOU: AUC, MCC, richiamo e precisione per classe.

Nel piano ROC i metodi pubblicati compaiono come punti di lavoro (HOU). Per DeepLoc, che sul test HPA pubblica solo
l'MCC, compaiono come curva dei punti con lo stesso MCC. Le AUC di DeepLoc sul test HPA sono solo nella Tabella S7 del
supplemento, non raggiungibile da qui: si aggiungono con un file. Resta dichiarato che test e training non coincidono
del tutto con quelli degli articoli.

## Statistica

**Un segnale delle varianti senza valori poteva risultare significativo?** Sì. Senza strutture, `contatti_sito` non
ha valori. Il bootstrap si fermava con un errore; nel caso limite, il p calcolato su ricampionamenti tutti NaN valeva
0,001, cioè "significativo". **→ corretto**: AUROC e p mancanti, verdetto "no".

**Il test HOU può fallire con un sottoinsieme vuoto?** Prima sì. **→ corretto**: un insieme con meno di 30
proteine non si valuta, e il notebook lo scrive.

## Omologia

**Il controllo con MMseqs2 deve essere obbligatorio?** Il progetto non deve dipendere da strumenti esterni. Il controllo
obbligatorio ora è dentro il notebook e più profondo:
- un terzo filtro di candidati, i profili di 3-meri pesati TF-IDF, che restano sensibili sotto il 40% di identità come
  il prefiltro di MMseqs2;
- per il test, 60 candidati per proteina invece di 8;
- la ricerca delle fughe nei due versi (aggiunte → test e test → aggiunte).

MMseqs2 resta una conferma facoltativa. Se il suo risultato c'è, ora si applica da solo: le aggiunte omologhe del test
escono dal training, e le proteine del test con un'omologa curata passano nello strato giusto. **→ corretto**.

## Limiti che restano (dichiarati, non eliminabili nel codice)
- Compartimenti neuronali: positivi e non annotati, nessun test esterno. Le probabilità sono un ordinamento.
- La rete `cervello` usata dallo 03 per le classi neuronali e gli ausiliari è a un seme. Per gli organelli lo 03 usa
  il modello scelto in CV.
- La replica di DeepLoc 2.0 è addestrata su proteine umane, non sulle 28.000 eucariote del modello pubblicato: va
  scritto "DeepLoc 2.0 riaddestrato sui nostri dati". Con circa 6.400 proteine i modelli ad attenzione possono essere
  svantaggiati: la conclusione vale a questa scala di dati.
- Wefers con il protocollo di DeepLoc non è il "Wefers ottimizzato" del loro articolo. È la loro architettura alla pari
  con le altre; il loro modello migliore non ha iperparametri pubblicati (non sono nel repository né su Zenodo).
