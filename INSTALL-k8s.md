# Installation on Kubernetes
Helm chart to deploy the Orchestrator solution suite on Kubernetes, including Janus IDP backstage, SonataFlow Operator, OpenShift Serverless Operator, Knative Eventing and Knative Serving.
The main focus of this chart is for development and quality testing on local or CI environments and it is not suited for production usage.
For that reason the images used for data-index and job-service are ephemeral.

This chart will deploy the following on the target OpenShift cluster:
  - Janus IDP backstage
  - SonataFlow Operator (with Data-Index and Job Service)
  - Knative Eventing
  - Knative Serving
  - serverless workflows 


## Prerequisites
- [Helm](https://helm.sh/docs/intro/install/) v3.9+ is installed.

For OpenShift-related configuration in the chart visit [here](https://github.com/bitnami/charts/blob/main/bitnami/postgresql/README.md#differences-between-bitnami-postgresql-image-and-docker-official-image).

## Installation

Build helm dependency and create a new project for the installation:
```console
git clone git@github.com:parodos-dev/orchestrator-helm-chart.git
cd orchestrator-helm-chart/charts
helm dep update orchestrator
oc new-project orchestrator
```

Inspect orchestrator/values-k8s.yaml in case you want to change localhost:9090 to something else.

Install the chart: 
```console
$ helm install orchestrator orchestrator -f orchestrator/values.yaml -f orchestrator/values-k8s.yaml 
```

When all the deployment is up backstage is available at localhost:9090. Kind can be configured to expose 
its ingress ports to the local machine (its simple port exposed by docker/podman) - see https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings
Alternatively you can port forward to localhost:

```console
kubectl port-forward svc/orchestrator-backstage 9090:7007 &
```


A sample output:
```
NAME: orchestrator
LAST DEPLOYED: Wed Feb  7 19:01:23 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Helm Release orchestrator installed in namespace default.


Components                   Installed   Namespace
====================================================================
Backstage                    YES        default
Postgres DB - Backstage      YES        default
Serverless Operator          YES        knative-serving     
SonataFlow Operator          YES        sonataflow-infra
KnativeServing               YES        knative-serving
KnativeEventing              YES        knative-eventing
SonataFlowPlatform           YES        sonataflow-infra
Data Index Service           YES        sonataflow-infra
Job Service                  YES        sonataflow-infra

Workflows deployed on namespace sonataflow-infra:
greeting

```
Run the following commands to wait until the workflow builds are done and workflows are running on namespace sonataflow-infra:
  oc wait -n sonataflow-infra sonataflow/greeting --for=condition=Built --timeout=15m
  oc wait -n sonataflow-infra sonataflow/greeting --for=condition=Running --timeout=5m
```


### Workflow installation

Follow [Workflows Installation](https://www.parodos.dev/serverless-workflows-helm/)

### Installation from OpenShift
```shell
cat << EOF | oc apply -f -
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: orchestrator
spec:
  connectionConfig:
    url: 'https://parodos-dev.github.io/orchestrator-helm-chart'
EOF
```

## Testing the Sample Workflow - Greeting

* Retrieve the route of the Greeting workflow service and save it environment variable $ROUTE.
```shell
$ ROUTE=`oc get route greeting -n sonataflow-infra -o=jsonpath='{.spec.host}'`
  echo $ROUTE
```
Sample output:
```
greeting-sonataflow-infra.apps.ocp413.lab.local
```
* Trigger the greeting workflow and save the workflow id from the response in environment variable $WORKFLOW_ID.
```shell
curl -s -k -X POST -H 'Content-Type:application/json' -H 'Accept:application/json' -d '{ "language": "Spanish" }' 'https://'$ROUTE'/greeting' | jq
```
* Sample response
```
{
  "id": "9cb41281-f827-4d66-aaa8-76ca2d0fb9e0",
  "workflowdata": {
    "language": "Spanish",
    "greeting": "Saludos desde YAML Workflow, "
  }
}
```

## Cleanup
To remove the installation from the cluster, run:
```console
$ helm delete orchestrator
release "orchestrator" uninstalled
```
Note that the CRDs created during the installation process will remain in the cluster.


## Documentation
See [Helm Chart Documentation](./charts/orchestrator/README.md) for information about the values used by the helm chart.

This helm chart uses the [Janus-IDP backstage helm chart](https://github.com/janus-idp/helm-backstage) as a dependency subchart. See the [chart documentation](https://github.com/janus-idp/helm-backstage/blob/main/charts/backstage/README.md) on how to override its values. 
