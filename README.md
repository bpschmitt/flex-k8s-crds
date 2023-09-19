# Monitor Kubernetes CRDs with New Relic Flex

In this repo, you'll find an example of how to use [New Relic Flex](https://github.com/newrelic/nri-flex) to monitor Kubernetes CRDs.  While CRD monitoring is not currently built into the [nri-kubernetes](https://github.com/newrelic/nri-kubernetes) integration, you can still enable CRD monitoring using [nri-flex](https://github.com/newrelic/nri-flex) and a custom flex config.  

How does it work?  Here's the TLDR:

- The deployment runs the `newrelic/infrastructure-bundle` container image which has `nri-flex` pre-packaged
- The running Infra agent is configured in "forward only" mode.  With this option, the agent will run the `nri-flex` integration, but not report host-level metrics.
- The `nri-flex` config curls the Kubernetes API for the defined CRDs and parses the output with the built-in `jq` parser


## Setup

First, add the example CRD to your cluster and create two `DemoWidget` resources.

```
kubectl create ns flex-demo
kubectl apply -f crds/. -n flex-demo
```

Next, deploy the New Relic agent.  Be sure to add your New Relic Ingest License Key as a secret.

```
kubectl create secret generic newrelic-license-key -n flex-demo --from-literal=license=<YOUR NEW RELIC LICENSE KEY>
kubectl apply -f agent/. -n flex-demo
```

That's it!  You should now see the `nri-flex-crd-monitor` pod running in the `flex-demo` namespace.

```
$ kubectl get pods -n flex-demo
NAME                                    READY   STATUS    RESTARTS   AGE
nri-flex-crd-monitor-7c457b5cbd-nqgdt   1/1     Running   0          171m
```

## Validating with NRQL

If running successfully, you should see two `DemoWidget` objects returned by the following NRQL query.

```
from K8sCRDSample select latest(enabled), if(latest(status) = 'UP', '🟢', '🔴') as 'Status', latest(kind), latest(apiVersion) facet name 
```

![New Relic](https://p191.p3.n0.cdn.getcloudapp.com/items/12uQ6GJq/092c8ce4-3393-465b-89d0-cfa705c64607.jpg?v=a2a00427366a0bec507c3124c26e9295)


## Manual Testing and Troubleshooting

You can manually run the `nri-flex` binary and your Flex config by opening a shell to the `flex` container.  Your command should look something like this.  Your pod name will be slightly different so be sure to update it.

```
kubectl exec -it nri-flex-crd-monitor-7c457b5cbd-nqgdt -n flex-demo -- /bin/bash
```

Once you have a shell running, set some ENV vars which will save you some typing.

```
export APISERVER=https://kubernetes.default.svc.cluster.local \
export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
export CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Curl Examples

Before running your Flex config manually, it's good idea to test with curl first to ensure there aren't any permissions issues that need to be sorted out.

This curl command should work.
```
curl $APISERVER/apis/example.com/v1/demowidgets --header "Authorization: Bearer $TOKEN" --cacert $CERT
```

This one should fail with a permissions issue.  Feel free to update the agent `ClusterRole` and `ClusterRoleBinding` if different RBAC is needed.

```
curl $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $TOKEN" --cacert $CERT
```

### Manually running Flex

You can use the following command from inside the container to test the Flex config manually.

```
export PATH=$PATH:/var/db/newrelic-infra/newrelic-integrations/bin && nri-flex -verbose -config_path /etc/newrelic-infra/integrations.d/k8s-crd.yaml -pretty
```

Your output should look similar to this.  If it doesn't, there should be errors in the output.  Use them as your guide to determining what the issue is (more than likely, a syntax issue in the ConfigMap).

```
# export PATH=$PATH:/var/db/newrelic-infra/newrelic-integrations/bin && nri-flex -verbose -config_path /etc/newrelic-infra/integrations.d/k8s-crd.yaml -pretty
DEBU[0000] Function.isAvailable: enter
DEBU[0000] Function.isAvailable: exit status: false
INFO[0000] com.newrelic.nri-flex                         GOARCH=amd64 GOOS=linux version=1.9.1
DEBU[0000] config: git sync configuration not set
WARN[0000] config: testing agent config, agent features will not be available
DEBU[0000] config: running async                         name=K8sCRDSample
DEBU[0000] config: processing apis                       apis=1 name=K8sCRDSample
DEBU[0000] fetch: collect data                           name=K8sCRDSample
DEBU[0000] commands: executing                           count=1 name=K8sCRDSample
DEBU[0000] processor-data: running data handler          name=K8sCRDSample
DEBU[0000] config: finished variable processing apis     apis=1 name=K8sCRDSample
INFO[0000] flex: completed processing configs            configs=1
{
	"name": "com.newrelic.nri-flex",
	"protocol_version": "3",
	"integration_version": "1.9.1",
	"data": [
		{
			"metrics": [
				{
					"apiVersion": "example.com/v1",
					"enabled": "true",
					"event_type": "K8sCRDSample",
					"integration_name": "com.newrelic.nri-flex",
					"integration_version": "1.9.1",
					"kind": "DemoWidget",
					"name": "my-demowidget",
					"status": "UP"
				},
				{
					"apiVersion": "example.com/v1",
					"enabled": "true",
					"event_type": "K8sCRDSample",
					"integration_name": "com.newrelic.nri-flex",
					"integration_version": "1.9.1",
					"kind": "DemoWidget",
					"name": "my-demowidget-two",
					"status": "DOWN"
				},
				{
					"event_type": "flexStatusSample",
					"flex.Hostname": "nri-flex-crd-monitor-7c457b5cbd-nqgdt",
					"flex.IntegrationVersion": "1.9.1",
					"flex.IsKubernetes": "true",
					"flex.counter.ConfigsProcessed": 1,
					"flex.counter.EventCount": 2,
					"flex.counter.EventDropCount": 0,
					"flex.counter.K8sCRDSample": 2,
					"flex.time.elapsedMs": 33,
					"flex.time.endMs": 1695088258284,
					"flex.time.startMs": 1695088258251
				}
			],
			"inventory": {},
			"events": []
		}
	]
}
```


