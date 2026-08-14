# BigDataToolsAnalysis

Analisi comparativa di prestazioni su stack Big Data, lungo due dimensioni:

- **Motore di elaborazione** — Hadoop MapReduce, Apache Hive e Apache Spark
- **Formato di archiviazione** — CSV, JSON, Avro, ORC e Parquet

Le stesse due analisi sono state implementate su ogni motore ed eseguite su dataset di dimensioni crescenti, misurando i tempi al variare sia del motore sia del formato in cui i dati sono archiviati.

## Obiettivo

Isolare l'impatto delle due variabili sulle prestazioni. Da un lato il motore, a parità di logica applicativa e di dati; dall'altro il formato di archiviazione, per confrontare formati testuali riga-orientati (CSV, JSON) con formati binari (Avro, riga-orientato) e colonnari (ORC, Parquet) — questi ultimi attesi come vantaggiosi su query che leggono un sottoinsieme di colonne, grazie a compressione e column pruning.

## Dataset

[Amazon Fine Food Reviews](https://www.kaggle.com/datasets/snap/amazon-fine-food-reviews) (Stanford Network Analysis Project) — circa 568.454 recensioni di prodotti alimentari raccolte tra il 1999 e il 2012, con identificativo prodotto e utente, punteggio, timestamp, testo della recensione e voti di utilità.

Il dataset non è incluso nel repository per via delle dimensioni: scarica `Reviews.csv` da Kaggle e posizionalo nella root del progetto prima di eseguire gli script.

### Aumento artificiale del volume

Il dataset originale, da solo, non è abbastanza grande per far emergere differenze significative tra i tre motori e tra i formati di archiviazione. È stato quindi ampliato generando righe fittizie a partire da quelle reali: ogni record viene replicato più volte con identificativo progressivo e variazione casuale dei campi numerici, mantenendo intatta la struttura e la distribuzione dei dati.

Questo consente di ottenere versioni del dataset di dimensioni crescenti e di osservare, al variare del volume:

- come cambia lo **spazio occupato su disco** dai cinque formati di archiviazione, e quindi il guadagno reale di compressione dei formati binari e colonnari rispetto a CSV e JSON
- come varia la **velocità di elaborazione** delle stesse query su MapReduce, Hive e Spark

La replica è gestita da `aumenta_dimensioni.py`, dove il parametro `num_copies` controlla il fattore di crescita.

## Analisi implementate

**1. Top prodotti per anno e parole più frequenti**
Per ogni anno, i 10 prodotti con più recensioni e, per ciascuno, le 5 parole più ricorrenti nel testo (escluse quelle sotto i 4 caratteri).

**2. Apprezzamento medio degli utenti**
Per ogni utente, il rapporto medio tra voti di utilità ricevuti e voti totali, con classifica decrescente.

## Formati di archiviazione

Lo stesso dataset è stato materializzato in cinque formati e riutilizzato per le medesime query:

| Formato | Tipo | Note |
|---|---|---|
| CSV | testuale, riga | formato di partenza, nessuna compressione |
| JSON | testuale, riga | schema ripetuto a ogni record |
| Avro | binario, riga | schema esplicito, buono in scrittura |
| ORC | binario, colonnare | compressione e indici interni |
| Parquet | binario, colonnare | compressione e column pruning |

Per ogni combinazione sono stati registrati tempo di esecuzione e spazio occupato su HDFS.

## Ambiente

Cluster locale con HDFS e YARN, Hive per l'accesso SQL e Spark in modalità standalone. Gli script MapReduce girano via Hadoop Streaming.

## Struttura del repository

| File | Motore | Analisi |
|---|---|---|
| `mapper.py`, `reducer.py` | MapReduce | Top prodotti e parole |
| `mapper2.py`, `reducer2.py` | MapReduce | Apprezzamento utenti |
| `top_products.hql` | Hive | Top prodotti e parole |
| `apprezzamento_utenti.hql` | Hive | Apprezzamento utenti |
| `top_products.py` | Spark | Top prodotti e parole |
| `apprezzamento_utenti.py` | Spark | Apprezzamento utenti |
| `aumenta_dimensioni.py` | — | Replica il dataset per i test di scalabilità |
| `elimina_campi1.py`, `elimina_campi2.py` | — | Preparazione dei dataset ridotti per le due analisi |
| `Grafici.xlsx` | — | Tempi di esecuzione misurati e grafici di confronto |

## Esecuzione

Preparazione dei dati:

```bash
python aumenta_dimensioni.py     # Reviews.csv -> Reviews_augmented.csv
python elimina_campi1.py         # -> Reviews_processed.csv (analisi 1)
python elimina_campi2.py         # -> Reviews_cleaned.csv   (analisi 2)
```

MapReduce (Hadoop Streaming):

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -files mapper.py,reducer.py \
  -mapper mapper.py -reducer reducer.py \
  -input /input/Reviews_processed.csv \
  -output /output/top_products
```

Hive:

```bash
hive -f top_products.hql
```

Spark:

```bash
spark-submit top_products.py \
  --input_path Reviews_processed.csv \
  --output_path output/top_products
```

## Risultati

I tempi di esecuzione misurati per ogni combinazione di motore, formato di archiviazione e dimensione del dataset sono raccolti in `Grafici.xlsx`, insieme allo spazio occupato dai singoli formati.

## Contesto

Progetto realizzato nell'ambito del corso di Big Data, Università degli Studi Roma Tre (2022).
