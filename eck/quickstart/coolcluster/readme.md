# Coolcluster

Defines multi-node Elasticsearch cluster. 

With the nginx-ingress, nginx will be able to recieve and pass through requests to the Elasticsearch ClusterIP service (the one created automatically by ECK Operator). Using `minikube tunnel` will start a tunnel for the NGINX Ingress and you will be able to reach your cluster on http://localhost/elasticsearch