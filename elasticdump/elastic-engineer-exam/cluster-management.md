# Elastic Engineer Exam: Cluster Management

## High availability cluster

__Cross cluster replication__ - CCR - innebär att ett _follower cluster_ håller en kopiera på data från _leader cluster_. Follwer-klustret står redo att ta över om något skulle gå snett för leader-klustret, samt ta hand om search-request om klienten är närmare det datacenter där follower-klustret finns.

__Snapshots__ är backups av ett Elasticsearchkluster. Dessa kan användas för att 
- skapa backups utan downtime
- återskapa data vid förlust
- överföra data till ett annat kluster
- reducera kostnad för storage genom att använda searchable snapshots för data i cold eller frozen data tiers

### Remote cluster setup

På det lokala klustret, se till att alla *master_nodes* även har rollen *remote_cluster_client*. 

För att addera ett remote cluster behöver man konfigurera det lokala klustret enligt följande:

PUT /_cluster/settings
{
  "persistent" : {
    "cluster" : {
      "remote" : {
        // Namnet på klustret måste stämma överens med det faktiska namnet på klustret
        "single-node-cluster" : { 
          "seeds" : [
            "elasticsearch04:9300"
          ]
        }
      }
    }
  }
}

Verifiera att klustret är connected:

GET _remote/info

### Cross cluster replication

_CCR är en betalningsfeature och kan därför inte utföras med basic license._

Efter att ett remote cluster är connected, konfigurera roller med privilegier för respektive kluster.

__Remote:__

curl -X POST "localhost:9200/_security/role/remote-replication?pretty" -H 'Content-Type: application/json' -d'
{
  "cluster": [
    "read_ccr"
  ],
  "indices": [
    {
      "names": [
        "leader-index-name"
      ],
      "privileges": [
        "monitor",
        "read"
      ]
    }
  ]
}
' -u elastic

Passa på att skapa ett index:

curl -X PUT "localhost:9200/my-remote-index/_doc/1" -H 'Content-Type: application/json' -d'
{
  "title": "Created on remote cluster"
}
' -u elastic

__Local:__

POST _security/role/remote-replication
{
  "cluster": [
    "manage_ccr"
  ],
  "indices": [
    {
      "names": [
        "follower-index-name"
      ],
      "privileges": [
        "monitor",
        "read",
        "write",
        "manage_follow_index"
      ]
    }
  ]
}

När rollerna är skapade, skapa en user på lokala klustret med rollen remote-replication:

POST _security/user/ccr-user
{
  "password": "password",
  "roles": [
    "remote-replication"
  ]
}

Och ett follower-index som pekar ut remote-klustret och leader-index:

PUT my-follower-index/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster": "single-node-cluster",
  "leader_index": "my-remote-index"
}

För att kontrollera status finns det ett get-follower-stats-API: 

GET <index>/_ccr/stats

Todo: 
- [ ] Tutorialen i dokumentation säger att follower index ska skapas på lokala klustret. ChatGPT håller med mig om att detta är bakvänt... Undersök detta

## Snapshot and restore

Snapshot: en backup av ett Elasticsearch kluster, förvarat i ett snapshot repository utanför klustret.

- Reguljära backups utan downtime
- Återställ förlorad data
- Migrera data till ett annat kluster
- Reducera storage costs genom _searchable snapshots_

Man kan använda cloudtjänster för storage, alternativt ett shared file system om man använder ett self-managed kluster på egen hårdvara. Genom _snapshot lifecycle management_ (SLM) kan man automatisera processen.

Per default innehåller ett snapshot följande:

- cluster state
  - persistent cluster settings
  - index templates, legacy index templates
  - ingest pipelines
  - ILM policies
  - feature states (data streams and indices used to store configurations, history etc)
- regular data streams och indices

De innehåller _inte_:

- transient cluster settings
- registered snapshot repositories
- node config files
- security config files

### Shared file system repository

1. Mount:a filsystemet till samma location på alla master- och datanoder
2. Sätt `path.repo` till filsystemets path i elasticsearch.yml och starta om varje nod
3. Använd _create snapshot repository API:et_ för att __registrera__ ett repo. Ange filsystemets path:

PUT _snapshot/my-repository
{
    "type": "fs",
    "settings": {
        "location": "/backup-repo"
    }
}

Verifiera:

POST _snapshot/my-repository/_verify

### Skapa en snapshot manuellt

Använd create snapshot-API:et för att ta en snapshot av klustet eller specifika data streams och index.

PUT _snapshot/my-repository/movies-snapshot
{
  "indices": "movies",
  "metadata": {
    "why": "Cause I enjoy great movies"
  }
}

Kontrollera status:

GET _snapshot/my-repository/movies-snapshot

### Restore:a ett index eller en data stream

För att undvika konflikter, ta bort det index/data stream som ska återställas. Ange sedan vilka index som ska återställas:

POST _snapshot/my-repository/movies-snapshot/_restore
{
  "indices": "movies",
  "include_global_state": true // false by default
}

Utan `include_global_state` inte index templates osv.

### Searchable snapshots

_Searchable snapshots är en betalningsfeature och kan därför inte utföras med basic license._

Cold och frozen data tiers använder searchable snapshots för att reducera storage- och operating costs. Searchable snapshots eliminerar behovet av replica shards.

Att söka i searchable snapshots är lika som för att söka ett vanligt index.

Vanligtvis använder man ILM för att att automatiskt konvertera ett vanligt index till ett searchable snapshot index när indexet når cold eller frozen phase. Man kan också göra ett index i ett existerande snapshot sökbart genom att manuellt mounta det med _mount snapshot API_:

POST _snapshot/my-repository/movies-snapshot/_mount
{
  "index": "movies",
  "renamed_index": "movies-searchable-snapshot", // The name of the index to create
  "index_settings": {
    "index.number_of_replicas": 0
  }
}

## Cross cluster search

Med CCS kan man köra en enstaka query mot ett eller flera remote clusters.

Search, Async search, Multi search, Search templates etc är supportade.

Se till att remote-klustret är anslutet:

PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "single-node-cluster": {
          "seeds": [
            "elasticsearch04:9300"
          ]
        }
      }
    }
  }
}

Verifiera:

GET _remote/info

Kör queries mot remote-klustret:

GET single-node-cluster:my-remote-index/_search

eller mot både lokala och remote-klustret:

GET single-node-cluster:my-remote-index,movies/_search
