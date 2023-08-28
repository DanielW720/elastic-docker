# Elastic Engineer Exam: Data Processing

## Analyzers

### Anatomi av en analyzer

- Noll eller flera _charachter filters_: Tar emot orginaltexten som en stream av chars och transformerar den genom add/remove/change.
- Exakt en _tokenizer_: Bryt upp texten till en stream av individuella tokens
- Noll eller flera _token filters_: Tar emot en stream av tokens av utför add/remove/change, exempelvis lowercase, stop, synonym

### Index vs Search analyzers

Textfält analyzeras både vid index time och query time. Oftast vill man använda samma analyzer vid index/query av samma index och fält. Det finns dock tillfällen då man vill använda en annan search analyzer.

Exempel:

Säg att vi vill söka på prefix. Ordet "apple" indexeras och tokeniseras till [a, ap, app, appl, apple]. Om vi använder samma analyzer vid search och söker på "appli" får vi matchingar eftersom "appli" tokeniseras till [a, ap, app, appl, appli]. Istället använder vi en search analyzer som tokeniserar "appli" till [appli].

Man kan sätta search analyzer på olika sätt, exempelvis vid query time, i ett specifikt fälts mappning, eller en default search analyzer för ett index genom `analysis.analyzer.default_search`. Sätter man en default search analyzer måste man också sätta en default index analyzer genom `analysis.analyzer.default`.

### Custom analyzers

_analyze kan användas för att testa en analyzer, exemelvis

POST _analyze
{
  "analyzer": "whitespace",
  "text":     "The quick brown fox."
}

eller, när en custom analyzer är definierad för ett index,

POST myindex/_analyze
{
  "analyzer": "custom-analyzer",
  "text":     "The quick brown fox."
}

Definiera en ny analyzer i myindex:

PUT myindex
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_analyzer": {
          "type": "custom", // **Detta är en custom analyzer. Vi hade också kunnat basera den på exempelvis standard analyzer genom att sätta "standrad" istället för "custom"**
          "char_filter": [],
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  }
}

Vi kan också definiera ett custom char_filter som byter ut en specifik följd av characters till något annat:


PUT myindex
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_analyzer": {
          "char_filter": [
            "arrow_translation"
          ],
          "tokenizer": "punctuation",
          "filter": [
            "lowercase"
          ]
        }
      },
      "char_filter": {
        "arrow_translation": {
          "type": "mapping",
          "mappings": [
            "-> => _right-arrow_",
            "<- => _left-arrow_"
          ]
        }
      },
      "tokenizer": {
        "punctuation": {
          "type": "pattern",
          "pattern": "[ .,!?|]"
        }
      }
    }
  }
}

POST myindex/_analyze
{
  "analyzer": "custom_analyzer",
  "text": "We're going to the -> and then to the <-"
}

Output tokens blir [we'er, going, to, the, _right-arrow_, and, then, to, the, _left-arrow_].

__Normalizers__ liknar analyzers, men avger endast ett token. De godtar inte tokenizers, utan endast char filters och token filters som behandlar en char i taget (t ex lowercase filter, men inte stemming filter). De används för keyword-fält för att åstadkomma tillåta att exempelvis "foo": "BÀR" ger träff på "foo": "bàr" trots att "foo" är av typen keyword. Dessutom innebär det att aggregeringar använder normaliserade värden, så två dokument med "foo": "bàr" respektive "foo": "BÀR" skulle returnera en term bucket för "foo": "bar" med 2 dokument.

Todo: 

- [ ] __ngram och edge-ngrams__

### Ingest pipeline with Painless scripts

Ingest pipelines låter dig transformera data innan indexering. En ingest pipeline kan exempelvis ta bort fält, extrahera värden från text och berika data.

Med ingest-API:et skapar man pipelines genom följande query:

PUT _ingest/pipeline/top-movies-pipeline
{
  "description": "Only includes movies with a score eqaul or higher than 7.0",
  "processors": [
    {
      "drop": {
        "if": "ctx.IMDB_rating < 7.0"
      }
    }
  ]
}

Testa pipelinen genom _simulate:

POST _ingest/pipeline/top-movies-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "IMDB_rating": 7.2
      }
    },
    {
      "_source": {
        "IMDB_rating": 6
      }
    }
  ]
}

Tillbaka får man dokumentet efter att det gått igenom pipelinen. 

_simulate kan även köras genom att definiera pipelinen direkt:

POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "drop": {
          "if": "ctx.IMDB_rating < 7.0"
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "IMDB_rating": 7.2
      }
    },
    {
      "_source": {
        "IMDB_rating": 6
      }
    }
  ]
}

__Applicera vid indexering__

Använd pipelinen vid indexering genom att sätta pipeline-parametern:

POST main/_doc/1?pipeline=top-movies-pipeline
{
  "IMDB_rating": 8.1
}

_pipeline_ kan även appliceras med _reindex-APIet:

POST _reindex
{
  "source": {
    "index": "movies"
  },
  "dest": {
    "index": "top-movies",
    "pipeline": "top-movies-pipeline"
  }
}

Todo:

- [ ] Pipeline fungerar ej med _reindex?

Default pipeline kan sättas med `index.default_pipeline` i indexets settings.

