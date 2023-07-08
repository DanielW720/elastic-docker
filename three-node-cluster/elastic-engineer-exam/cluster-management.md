# Elastic Engineer Exam: Cluster Management

## High availability cluster

__Cross cluster replication__ - CCR - innebär att ett _follower cluster_ håller en kopiera på data från _leader cluster_. Follwer-klustret står redo att ta över om något skulle gå snett för leader-klustret, samt ta hand om search-request om klienten är närmare det datacenter där follower-klustret finns.

__Snapshots__ är backups av ett Elasticsearchkluster. Dessa kan användas för att 
- skapa backups utan downtime
- återskapa data vid förlust
- överföra data till ett annat kluster
- reducera kostnad för storage genom att använda searchable snapshots för data i cold eller frozen data tiers

### CCR setup

_CCR är en betalningsfeature och kan därför inte utföras med basic license._

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

Efter detta behöver man konfigurera roller med privilegier för respektive kluster.

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

POST /_security/role/remote-replication
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

För att kontrollera statu finns det ett get-follower-stats-API: 

GET <index>/_ccr/stats

Todo: 
- [ ] Tutorialen i dokumentation säger att follower index ska skapas på lokala klustret. ChatGPT håller med om att detta är bakvänt... Undersök detta