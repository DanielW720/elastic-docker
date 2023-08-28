# Elastic Engineer Exam: Data Management

Följande serie av requests skapar 

- En simpel ILM policy 
- En component template som definierar settings (ink. en ILM policy) och mappning. Mappningen inkluderar ett @timestamp-fält, vilket är nödvändigt för att skapa en data stream.
- En index template som använder föregående component template. Skapar även readonly- och readwrite-alias automatiskt

För data kan man ladda in kibana_sample_data_ecommerce och omindexera till ecommerce-<någonting>.

## Todo:

- [ ] Gör om detta för repetition. Använd Sample web logs istället eftersom data streams är designade för use-cases så som när existerande data sällan, om ens någonsin, uppdateras. Exempelvis eventdata, statistik och loggar.
- [ ] Placera data i en searchable snapshot vid cold phase/frozen phase.

* Note: trots att jag satte policyn till att övergå till warm phase efter 1 min tog det cirka 5-10 minuter innan det skedde

### PUT _ilm/policy/ecommerce-lp
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1m",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          }
        }
      }
    }
  }
}

### PUT _component_template/ecommerce_component_template
{
  "template": {
    "settings": {
      "index": {
        "number_of_shards": "3",
        "number_of_replicas": "2"
      },
      "index.lifecycle.name": "ecommerce-lp"
    },
    "mappings": {
      "dynamic_templates": [
        {
          "custom_title_template": {
            "mapping": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword"
                }
              }
            },
            "match": "title-*"
          }
        }
      ],
      "properties": {
        "type": {
          "type": "keyword"
        },
        "manufacturer": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "products": {
          "properties": {
            "tax_amount": {
              "type": "half_float"
            },
            "taxful_price": {
              "type": "half_float"
            },
            "quantity": {
              "type": "integer"
            },
            "taxless_price": {
              "type": "half_float"
            },
            "discount_amount": {
              "type": "half_float"
            },
            "base_unit_price": {
              "type": "half_float"
            },
            "discount_percentage": {
              "type": "half_float"
            },
            "product_name": {
              "analyzer": "english",
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword"
                }
              }
            },
            "manufacturer": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword"
                }
              }
            },
            "min_price": {
              "type": "half_float"
            },
            "created_on": {
              "type": "date"
            },
            "price": {
              "type": "half_float"
            },
            "unit_discount_amount": {
              "type": "half_float"
            },
            "product_id": {
              "type": "long"
            },
            "base_price": {
              "type": "half_float"
            },
            "_id": {
              "type": "text",
              "fields": {
                "keyword": {
                  "ignore_above": 256,
                  "type": "keyword"
                }
              }
            },
            "category": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword"
                }
              }
            },
            "sku": {
              "type": "keyword"
            }
          }
        },
        "customer_last_name": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "day_of_week_i": {
          "type": "integer"
        },
        "total_quantity": {
          "type": "integer"
        },
        "currency": {
          "type": "keyword"
        },
        "taxless_total_price": {
          "type": "half_float"
        },
        "total_unique_products": {
          "type": "integer"
        },
        "event": {
          "properties": {
            "dataset": {
              "type": "keyword"
            }
          }
        },
        "sku": {
          "type": "keyword"
        },
        "email": {
          "type": "keyword"
        },
        "day_of_week": {
          "type": "keyword"
        },
        "geoip": {
          "properties": {
            "continent_name": {
              "type": "keyword"
            },
            "city_name": {
              "type": "keyword"
            },
            "country_iso_code": {
              "type": "keyword"
            },
            "location": {
              "type": "geo_point"
            },
            "region_name": {
              "type": "keyword"
            }
          }
        },
        "customer_first_name": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "customer_phone": {
          "type": "keyword"
        },
        "customer_birth_date": {
          "type": "date"
        },
        "customer_full_name": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "order_date": {
          "type": "date"
        },
        "@timestamp": {
          "type": "date"
        },
        "category": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "customer_id": {
          "type": "keyword"
        },
        "order_id": {
          "type": "keyword"
        },
        "user": {
          "type": "keyword"
        },
        "customer_gender": {
          "type": "keyword"
        },
        "taxful_total_price": {
          "type": "half_float"
        }
      }
    }
  }
}


### PUT _index_template/ecommerce_index_template
{
  "index_patterns": [
    "ecommerce-*"
  ],
  "template": {
    "aliases": {
      "ecommerce-readonly": {
        "is_write_index": false
      },
      "ecommerce-readwrite": {}
    }
  },
  "priority": 500,
  "composed_of": [
    "ecommerce_component_template"
  ],
  "_meta": {
    "description": "Template for my ecommerce indices and data streams",
    "myfield": "Useless field :)"
  }
}


