Kopiera elastic-certificates.p12 till det lokala filsystemet:

`docker cp <container>:"/usr/share/elasticsearch/elastic-certificates.p12" "/c/Users/westedan/repositories/elastic-docker/three-node-cluster/node-configs/tls/elastic-certificates.p12"`

Kopiera elastic-certificates.p12 till en container:

`docker cp ./elastic-certificates.p12 <container>:"/usr/share/elasticsearch/config/elastic-certificates.p12"`

