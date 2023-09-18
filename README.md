# flex-k8s-crds


In this repo, you'll find an example of how to use New Relic Flex to monitor Kubernetes CRDs.

## Setup

First, add the example CRD to your cluster.

```
kubectl create ns flex-demo
kubectl apply -f crd/. -n flex-demo
```

Next, deploy the New Relic agent.

```
kubectl create secret generic newrelic-license-key -n flex-demo --from-literal=license=<YOUR NEW RELIC LICENSE KEY>
kubectl apply -f agent.yaml -n flex-demo
```

## Testing
```
export APISERVER=https://kubernetes.default.svc.cluster.local \
export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
export CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Curl Examples
```
curl $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $TOKEN" --cacert $CERT

curl $APISERVER/apis/example.com/v1/demowidgets --header "Authorization: Bearer $TOKEN" --cacert $CERT
```

### Manually running Flex

After opening a shell into the container, you can use the following command to test the config manually.

```
export PATH=$PATH:/var/db/newrelic-infra/newrelic-integrations/bin && nri-flex -verbose -config_path /etc/newrelic-infra/integrations.d/k8s-crd.yaml -pretty
```