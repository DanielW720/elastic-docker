# Elastic Cloud on Kubernetes Quickstart tutorial

Walkthrough av ECK: https://www.youtube.com/watch?v=KzBa7HXlT4g

## ECK Overview

Built on the Kubernetes Operator pattern, Elastic Cloud on Kubernetes (ECK) extends the basic Kubernetes orchestration capabilities to support the setup and management of Elasticsearch, Kibana, APM Server, Enterprise Search, Beats, Elastic Agent, Elastic Maps Server, and Logstash on Kubernetes.

With Elastic Cloud on Kubernetes you can streamline critical operations, such as:

- Managing and monitoring multiple clusters
- Scaling cluster capacity and storage
- Performing safe configuration changes through rolling upgrades
- Securing clusters with TLS certificates
- Setting up hot-warm-cold architectures with availability zone awareness

## Quickstart

With Elastic Cloud on Kubernetes (ECK) you can extend the basic Kubernetes orchestration capabilities to easily deploy, secure, upgrade your Elasticsearch cluster, and much more.

(Quickstart)[https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html]

### Start a K8s Cluster

E.g. a local K8s cluster such as [Minikube](https://minikube.sigs.k8s.io/docs/start/).

### Deploy ECK in your Kubernetes cluster

Installer CRDs:

`kubectl create -f https://download.elastic.co/downloads/eck/2.11.1/crds.yaml`

Install the operator with its RBAC rules:

`kubectl apply -f https://download.elastic.co/downloads/eck/2.11.1/operator.yaml`

Output:

```
namespace/elastic-system created
serviceaccount/elastic-operator created
secret/elastic-webhook-server-cert created
configmap/elastic-operator created
clusterrole.rbac.authorization.k8s.io/elastic-operator created
clusterrole.rbac.authorization.k8s.io/elastic-operator-view created
clusterrole.rbac.authorization.k8s.io/elastic-operator-edit created
clusterrolebinding.rbac.authorization.k8s.io/elastic-operator created
service/elastic-webhook-server created
statefulset.apps/elastic-operator created
validatingwebhookconfiguration.admissionregistration.k8s.io/elastic-webhook.k8s.elastic.co created
```

The operator along with its service account and other resources are installed in the newly created `elastic-system` namespace. It is recommended that you choose a dedicated namespace for your workloads, rather than using the elastic-system or the default namespace.

Monitor the operator logs:

`kubectl -n elastic-system logs -f statefulset.apps/elastic-operator`

### Deploy an Elasticsearch cluster

Ensure your cluster has at least one node with at least 2gb of free memory, else the pods will be stuck in *Pending* state.

The operator automatically creates and manages Kubernetes resources to achieve the desired state of the Elasticsearch cluster. It may take up to a few minutes until all the resources are created and the cluster is ready for use.

#### Single Node Cluster

**Elasticsearch setup:**

Create a dedicated namespace for your cluster:

`kubectl create namepspace elastic-single-node`

Use the cluster:

`kubectl config set-context --current --namespace=elastic-single-node`

Apply [single-node/elasticsearch.yml](./single-node/elasticsearch.yml) to deploy a single node Elasticsearch cluster:

`kubectl apply -f ./single-node/elasticsearch.yml`

Get an overview of the cluster:

`kubectl get elasticsearch`

A ClusterIP Service has been automatically created. Port-forward to reach it on your local machine outside the K8s cluster:

`kubectl port-forward service/quickstart-es-http 9200`

Since security is turned on by default for Elastic v. 8, you're going to need the password for the default user "elastic". 

`kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'`

Copy the password.

HTTPS is already set up with self-signed certificate. When using Curl, the flag `-k` can be used to disable certificate verification:

`curl -u "elastic:<PASSWORD>" -k "https://localhost:9200"`

**Kibana setup:**

Create a Kibana instance:

`kubectl apply -f ./single-node/kibana.yml`

Again, a ClusterIP Service is created automatically. Port-forward it:

`kubectl port-forward service/quickstart-kb-http 5601`

Open https://localhost:5601 in your browser. Your browser will show a warning because the self-signed certificate configured by default is not verified by a known certificate authority and not trusted by your browser. You can temporarily acknowledge the warning for the purposes of this quick start but it is highly recommended that you configure valid certificates for any production deployments.

Get the password for the `elastic` user:

`kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo`
