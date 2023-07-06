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

