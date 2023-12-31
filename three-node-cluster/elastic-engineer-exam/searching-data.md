# Elastic Engineer Exam: Searching Data

## Async search

Följande request utför en async search request. "_async_search" accepterar allting som "_search" accepterar.

POST ecommerce-readonly/_async_search
{
  "size": 0,
  "aggs": {
    "orders_per_weekday": {
      "terms": {
        "field": "order_date_weekday",
        "size": 10,
        "order": {
          "_count": "desc"
        }
      }
    }
  }
}

Om requesten ovan går väldigt snabbt (under 1 sekund per default) så sparas inte responsen. Med paramtern keep_on_completion=true sparas den ändå (i 5 dagar per default) och requestens id returneras. `GET _async_search/<id>` kan användas för att hämta resultatet. 

## Aggregations (bucket och metric)

- Bucket aggregations: skapar buckets med dokument. Varje bucket är associerad med ett kriterie som bestämmer om ett dokument i nuvarna kontexten ska "tas med". Bucket aggregations kan till skillnad från metric aggregations innehålla sub-aggregations (makes sense; varje bucket är ett document set!).
- Metric aggregations: Beräknar metrics baserade på de extraherade värdena från de dokument som aggregeras på. De extraherade värdena kan är vanligtvis från dokument-fälten, men kan också genereras med script.

Följande query använder både ett runtime field och en sub-aggregation på runtime-fältet. 

GET ecommerce-readonly/_search
{
  "size": 1,
  "_source": false, 
  "fields": [
    "products_price_sum"
  ], 
  "runtime_mappings": {
    "products_price_sum": {
      "type": "double",
      "script": """
      double sum = 5.5;
      emit(sum);
      """
    }
  },
  "aggs": {
    "orders_per_weekday": {
      "terms": {
        "field": "order_date_weekday",
        "size": 7,
        "order": {
          "_count": "desc"
        }
      },
      "aggs": {
        "max_price": {
          "max": {
            "field": "products_price_sum"
          }
        }
      }
    }
  }
}

Notera att fältet fields används för att returnera runtime-fältet. Runtime fields returneras aldrig i _source.

Todo: 

- [ ] Gör något annat i scriptet

### Mer avancerat exempel:

Aggregation query som plockar ut 2000-talets filmer med Meta_score bättre eller lika med 80, och listar de filmer med bäst rating per år. Inkluderar också medelrating för varje år. För att aggregationen endast ska returnera en enda titel behöver vi sätta size=1 i title-term. Men vi vill också att denna titel ska vara den med högst rating, därför behöver vi plocka ut ratingen för varje titel och sedan sortera på det värdet med bucket_sort (pipeline aggregation).

GET imdb-movies/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "Released_Year": {
              "gte": 2000
            }
          }
        },
        {
          "range": {
            "Meta_score": {
              "gte": 80
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "year-histogram": {
      "histogram": {
        "field": "Released_Year",
        "interval": 1,
        "order": {
          "_key": "desc"
        }
      },
      "aggs": {
        "avg-imdb-rating": {
          "avg": {
            "field": "IMDB_Rating"
          }
        },
        "title-terms": {
          "terms": {
            "field": "Series_Title.raw",
            "size": 1,
            "order": {
              "rating.value": "desc"
            }
          },
          "aggs": {
            "rating": {
              "max": {
                "field": "IMDB_Rating"
              }
            },
            "rating-sort": {
              "bucket_sort": {
                "sort": [
                  {
                    "rating": {
                      "order": "desc"
                    }
                  }
                ]
              }
            }
          }
        }
      }
    }
  }
}

## Runtime fields

Följande queries bygger ut mappningen av ecommerce-readwrite för att inkludera ett runtime field, och sedan använda fields API i _search för att hämta hem fältet vid search.

Det är inte nödvändigt att lägga in runtime field i mappningen, utan man kan även sätta "runtime_mappings" direkt i search requesten.

PUT ecommerce-readwrite/_mapping
{
  "runtime": {
    "order_date_weekday": {
      "type": "keyword",
      "script": {
        "source": "emit(doc['order_date'].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
      }
    }
  }
}

GET ecommerce-readonly/_search
{
  "_source": false,
  "fields": [
    "order_date_weekday",
    "category"
  ]
}
