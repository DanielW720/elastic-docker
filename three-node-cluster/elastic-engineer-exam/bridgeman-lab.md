Lösningar till labbövningarna av [George Bridgeman](https://georgebridgeman.com/exercises/)

# 19
GET olympic-events-fixed/_search
{
  "_source": [
    "event"
  ],
  "size": 0, 
  "query": {
    "match": {
      "event": "Gymnastics"
    }
  },
  "aggs": {
    "gymnastic_events": {
      "terms": {
        "field": "event.keyword",
        "size": 100
      }
    }
  }
}

# 20
GET olympic-events-fixed/_search
{
  "size": 0,
  "query": {
    "match": {
      "event": "Gymnastics"
    }
  },
  "aggs": {
    "genders": {
      "terms": {
        "field": "gender",
        "size": 10
      },
      "aggs": {
        "average_weight": {
          "avg": {
            "field": "weight"
          }
        }
      }
    }
  }
}

# 21
GET olympic-events-fixed/_search
{
  "size": 0,
  "aggs": {
    "events": {
      "terms": {
        "field": "event.keyword",
        "size": 600
      },
      "aggs": {
        "min-year": {
          "min": {
            "field": "year"
          }
        },
        "year-sort-asc": {
          "bucket_sort": {
            "sort": [
              {
                "min-year": {
                  "order": "asc"
                }
              }
            ]
          }
        }
      }
    }
  }
}

GET olympic-events-fixed/_search

# 22
GET olympic-events-fixed/_search
{
  "track_total_hits": true,
  "sort": [
    {
      "height": {
        "order": "desc"
      }
    }
  ],
  "size": 50,
  "_source": false,
  "fields": [
    "athleteName",
    "team",
    "sport",
    "age",
    "height",
    "weight",
    "gender","city"
  ],
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "year": {
              "value": 2016
            }
          }
        },
        {
          "term": {
            "city.keyword": "Rio de Janeiro"
          }
        }
      ]
    }
  }
}


# 23
GET olympic-events-fixed/_search
{
  "track_total_hits": true,
  "sort": [
    {
      "height": {
        "order": "desc"
      }
    }
  ],
  "size": 50,
  "_source": false,
  "fields": [
    "athleteName",
    "team",
    "sport",
    "age",
    "height",
    "weight",
    "gender",
    "weightLbs"
  ],
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "year": {
              "value": 2016
            }
          }
        },
        {
          "term": {
            "city.keyword": "Rio de Janeiro"
          }
        }
      ]
    }
  },
  "runtime_mappings": {
    "weightLbs": {
      "type": "double",
      "script": {
        "source": "emit(doc['weight'].value * 2.2);"
      }
    }
  }
}

# 24
GET olympic-events-fixed/_search
{
  "track_total_hits": true,
  "sort": [
    {
      "height": {
        "order": "desc"
      }
    }
  ],
  "size": 50,
  "_source": false,
  "fields": [
    "athleteName",
    "team",
    "sport",
    "age",
    "height",
    "weight",
    "gender",
    "weightLbs",
    "bmi"
  ],
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "year": {
              "value": 2016
            }
          }
        },
        {
          "term": {
            "city.keyword": "Rio de Janeiro"
          }
        }
      ]
    }
  },
  "runtime_mappings": {
    "weightLbs": {
      "type": "double",
      "script": {
        "source": "emit(doc['weight'].value * 2.2);"
      }
    },
    "bmi": {
      "type": "double",
      "script": {
        "source": """
        
        double weight = doc['weight'].value;
        double heightMeter = doc['height'].value / 100;
        double heightMeterSquared = heightMeter * heightMeter;
        emit(weight / heightMeterSquared);
        
        """
      }
    }
  }
}

# 25
GET olympic-events-fixed/_search
{
  "size": 50,
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ], 
  "query": {
   "term": {
     "medal": {
       "value": "Gold"
     }
   }
  }
}

# 26
GET olympic-events-fixed/_search
{
  "size": 1000,
  "fields": [
    "critera_matched"
  ],
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "weight": {
              "gt": 60,
              "lt": 70
            }
          }
        },
        {
          "range": {
            "age": {
              "lt": 20
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  },
  "runtime_mappings": {
    "critera_matched": {
      "type": "keyword",
      "script": {
        "source": """
        
        double weight = doc['weight'].value;
        long age = doc['age'].value;
        
        boolean isInWeightRange = (weight > 60) && (weight < 70);
        boolean isAgeUnderTwenty = (age < 20);
        
        if (isInWeightRange && isAgeUnderTwenty) {
          emit('Both');
        }
        else if (isInWeightRange) {
          emit('Weight');
        }
        else if (isAgeUnderTwenty) {
          emit('Age');
        }
        else {
          emit('None');
        }
        
        """
      }
    }
  }
}

# 29
PUT _enrich/policy/olympic-noc-append
{
  "match": {
    "indices": "olympic-noc-region",
    "match_field": "noc",
    "enrich_fields": [
      "notes",
      "region"
    ]
  }
}

PUT _enrich/policy/olympic-noc-append/_execute

PUT _ingest/pipeline/enrich-noc
{
  "processors": [
    {
      "enrich": {
        "policy_name": "olympic-noc-append",
        "field": "noc",
        "target_field": "nocDetails"
      }
    }
  ]
}

# 30
PUT olympic-events-enriched
{
  "mappings": {
    "dynamic": "true"
  }
}

# 31
POST _reindex
{
  "source": {
    "index": "olympic-events-fixed"
  },
  "dest": {
    "index": "olympic-events-enriched",
    "pipeline": "enrich-noc"
  }
}

GET olympic-events-enriched/_search

