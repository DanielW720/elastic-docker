# Elastic Engineer Exam: Developing Search Applications

## Highlighting

Följande query söker på shoes i två text-fält. Highlighting läggs på båda. Tags sätts till <mark></mark>, men specificeras om till <span></span> för `products.product_name`.

GET ecommerce-readonly/_search
{
  "query": {
    "multi_match": {
      "query": "shoes",
      "fields": [
        "category",
        "products.product_name"
      ]
    }
  },
  "highlight": {
    "fields": {
      "products.product_name": {
        "pre_tags": [
          "<span>"
        ],
        "post_tags": [
          "</span>"
        ]
      },
      "category": {}
    },
    "pre_tags": [
      "<mark>"
    ],
    "post_tags": [
      "</mark>"
    ]
  }
}

## Sort results 

GET ecommerce-readonly/_search
{
  "_source": false,
  "fields": [
    "order_date", 
    "category",
    "products.product_name"
  ],
  "sort": [
    {
      "category.keyword": {}
    },
    {
      "products.product_name.keyword": {}
    }
  ],
  "query": {
    "match": {
      "category": "shoes"
    }
  }
}

## Pagination

Paginering kan implementeras med `from`, dvs antalet hits att skippa, och `size` för att sätta antalet hits per sida. Detta är ej rekommenderat: https://www.elastic.co/guide/en/elasticsearch/reference/8.1/paginate-search-results.html

Använd istället `search_after`.

Eftersom refreshes kan ske mellan de separata pagineringsbläddringarna så kan ordningen rubbas. Börja därför med att skapa en point-in-time för att bevara nuvarande index state mellan sökningarna:

`POST ecommerce-readonly/_pit?keep_alive=10m`

Submitta sedan en sökning med ett sort-argument och speficicera PIT ID i pit.id parametern. Specificera inte något index. Den informationen kommer med i PIT ID:t.

GET /_search
{
  "size": 5,
  "query": {
    "match": {
      "products.product_name": "t-shirt"
    }
  },
  "pit": {
    "id": <pit-id>,
    "keep_alive": "10m" 😶‍🌫️*förnyar livslängden vid varje request*
  },
  "sort": [
    {
      "order_date": {
        "order": "desc"
      }
    }
  ]
}

Föregående sökning ger ett första resultat. För att gå till nästa sida, plocka ut sort-värdet och tiebreaker-värdet ur det n:te resultatet och lista dessa i fältet `search_after`, till exempel:

"search_after": [
    "Men's Accessories",
    214
],
"track_total_hits": false

`"track_total_hits": false` är rekommenderat för att göra sökningen snabbare.

## Index aliases

Skapa ett alias med read- och write-behörighet:

POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "ecommerce-20230705",
        "alias": "ecommerce-readwrite-2",
        "is_write_index": true
      }
    }
  ]
}

Testa att indexera ett dokument:

PUT ecommerce-readwrite-2/_doc/1
{
  "products": [
    {
      "product_name": "Party pants"
    }
  ]
}

Titta på dokumentet:

GET ecommerce-readwrite/_doc/1

Radera dokumentet:

DELETE ecommerce-readwrite/_doc/1

## Search template

Skapa en search template "my-search-template", som applicerar två parametriserade värden med nycklarna `user_query` och `user_size`, där den senare har ett defaultvärde 5:

PUT _scripts/my-search-template
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "category": "{{user_query}}"
        }
      },
      "size": "{{user_size}}{{^user_size}}5{{/user_size}}"
    }
  }
}

Titta på hur en request body ser ut:

POST _render/template
{
  "id": "my-search-template",
  "params": {
    "user_query": "hi",
    "user_size": 2
  }
}

Använd my-search-template:

GET ecommerce-readonly/_search/template
{
  "id": "my-search-template",
  "params": {
    "user_query": "shoes",
    "user_size": 1
  }
}