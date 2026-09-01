# PHP2xAI-sentiment
Sentiment analysis example built with PHP2xAI. The project demonstrates text preprocessing, embeddings, neural network training, and inference using the PHP frontend and C++ runtime. Sentiment labels are based on the RubixML Sentiment example and are fully acknowledged.


## Generare corpus e tokenizer

Eseguire i comandi dalla root del repository. Sono richiesti PHP 8.1 o superiore, le dipendenze Composer e il pacchetto PHP2xAI installato dentro vendor/antoniogweb/php2xai.

### 1. Generare corpus.txt

src/corpus.php raccoglie tutti i file .txt in src/train/ e src/test/, normalizza gli spazi e scrive un documento per riga:

```bash
php src/corpus.php
```

Il risultato è src/corpus.txt. Il corpus deve essere in UTF-8 e contenere testo normale, non ID di token o token speciali.

### 2. Generare tokenizer.json e tokenizer.meta.json

Il trainer PHP2xAI usa ByteLevel BPE e produce il tokenizer e i relativi metadati:

```bash
vendor/antoniogweb/php2xai/src/Tokenizer/Rust/Trainer/Bin/linux-x86_64/php2xai-tokenizer-trainer \
    --input src/corpus.txt \
    --output src/tokenizer.json \
    --vocab-size 30000 \
    --min-frequency 2
```

Se necessario, renderlo eseguibile:

```bash
chmod +x vendor/antoniogweb/php2xai/src/Tokenizer/Rust/Trainer/Bin/linux-x86_64/php2xai-tokenizer-trainer
```

Vengono creati:

```text
src/tokenizer.json       # vocabolario, merge BPE e configurazione
src/tokenizer.meta.json  # statistiche, incluso vocabulary_size
```

--vocab-size è un limite massimo. La dimensione effettiva va letta da tokenizer.meta.json:

```bash
php -r '$m=json_decode(file_get_contents("src/tokenizer.meta.json"), true, 512, JSON_THROW_ON_ERROR); echo $m["vocabulary_size"].PHP_EOL;'
```

Il valore vocabulary_size deve coincidere con la prima dimensione della tabella degli embedding in src/model.php (V). I token speciali sono già inclusi: non aggiungere +4.

### 3. Generare i dataset numerici

Dopo il tokenizer, convertire i documenti in ID e label:

```bash
php src/create_dataset.php
```

Il comando produce src/DataLabelInt/train.txt e src/DataLabelInt/test.txt. La sequenza è lunga 1024 token e in training e inferenza deve essere usato lo stesso tokenizer.json.

### Ordine completo

```bash
php src/corpus.php
vendor/antoniogweb/php2xai/src/Tokenizer/Rust/Trainer/Bin/linux-x86_64/php2xai-tokenizer-trainer \
    --input src/corpus.txt --output src/tokenizer.json \
    --vocab-size 30000 --min-frequency 2
php src/create_dataset.php
```


## Addestramento e validazione

Gli script `train.php` e `validate.php` devono essere eseguiti dalla directory
`src/`, perché usano percorsi relativi per `vendor/`, `DataLabelInt/` e gli
artefatti del modello.

### 4. Addestrare il modello

Prima di iniziare, assicurarsi di avere completato la generazione del tokenizer
e dei dataset e di avere installato le dipendenze Composer nella root del
progetto. Avviare quindi il training con:

```bash
cd src
php train.php
```

Il training usa:

- `DataLabelInt/train.txt` come dataset di addestramento;
- `DataLabelInt/test.txt` come dataset di validazione durante l'addestramento;
- batch da 300 campioni;
- 20 epoche;
- runtime C++ con provider Eigen.

Al termine vengono salvati nella directory `src/` gli artefatti del modello,
tra cui `weights.json` e `model.json`. Non eliminare o sostituire questi file
prima della validazione.

### 5. Validare il modello

La validazione ricarica `model.json` e `weights.json`, esegue l'inferenza sul
dataset `DataLabelInt/test.txt` e stampa il numero di campioni, le predizioni
corrette, l'accuracy e il tempo trascorso:

```bash
cd src
php validate.php
```

Esempio di output:

```text
Test samples: ...
Correct: ...
Accuracy: ... %
Elapsed: ... s
```

Per una nuova sessione di training è sufficiente rieseguire `php train.php`;
la validazione successiva userà i nuovi pesi salvati.
