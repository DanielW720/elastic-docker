# Elastic on K8s

Deploy an Elasticsearch cluster and one Kibana instance manually on K8s. This is an alternative to using the ECK Operator by Elastic.

## Start Minikube

Start Minikube: `minikube start`

View cluster in browser: `minikube dashboard`

## A few words about Helm

Using Helm (K8s package manager) let's us

- Simplify deployment
  - No need for manual configuration updates on individual objects, simply update the Helm Release to install the updates.
- Dynamic configuration
  - The `values.yml` file can be utilized to dynamically configure objects, or reference the same variable multiple times without unnecessary repetition.
- Easy upgrades and rollbacks
  - Helm keeps track of each release. Rollbacks are straightforward if a new configuration issue arise.
- Version control
  - Helm charts can be version controlled to track changes and view history of the configuration files.

## Deploying and port-forwarding

Change your current namespace to simplify Helm commands:

`kubectl config set-context --current --namespace <namespace>`

Install the Helm chart as a Helm release:

`helm install <release-name> <chart>`

If any changes are made to the chart, upgrade the release:

`helm upgrade <release-name> <chart>`

A simple way to access Kibana is to port-forward the Kibana ClusterIP service:

`kubectl port-forward service/kibana-service 5601`

Kibana should now be available on localhost:5601.


