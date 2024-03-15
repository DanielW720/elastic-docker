# Using Helm with ECK

Add repo:

`helm repo add elastic https://helm.elastic.co`

`helm repo update`

Since CRDs are global resources, they need to be installed by an administrator. This can be achieved by:

`helm install elastic-operator-crds elastic/eck-operator-crds`

The operator can be installed by any user who has full access to the set of namespaces they wish to manage. The following example installs the operator to elastic-system namespace and configures it to manage only the elastic namespace:

```
helm install elastic-operator elastic/eck-operator \
  -n elastic-system --create-namespace \
  --set=installCRDs=false \
  --set=managedNamespaces='{elastic}' \
  --set=createClusterScopedResources=false \
  --set=webhook.enabled=false \
  --set=config.validateStorageClass=false
```

To set up Elasticsearch and Kibana, check out [coolCluster](../coolcluster/) or [single-node](../single-node/).
