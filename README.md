# OpenShift Serverless - Demo notes

## Install & Hello World

### DEMO: Serverless

```yaml
# serverless-subscription.yaml
# OpenShift Serverless subcription
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: serverless-operator
  namespace: openshift-serverless
spec:
  channel: stable
  name: serverless-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Check:

```bash
oc get csv
```

### Deploy knative serving

```yaml
# knative-serving.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: knative-serving
---
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  high-availability:
    replicas: 1

```
Check:

```bash
oc get knativeserving knative-serving -n knative-serving
```

### Create ksvc

```bash
kubectl create ns demo
kubens demo
kn service create hello --image=quay.io/rhdevelopers/knative-tutorial-greeter:quarkus
http $(kn service list demo -o jsonpath='{.items[].status.url}')
kn service delete hello
```

## DEMO: Autoscalling

### Deploy service with limit

```bash
kn service create showcase \
  --image quay.io/cardil/knative-serving-showcase:2-send-event \
  --env DELAY=1500 \
  --concurrency-limit 1
```

Observe with `k9s`.

### Call with `hey`

```bash
http $(kn service list showcase -o jsonpath='{.items[].status.url}')/hello
hey -c 10 -n 200 $(kn service list showcase -o jsonpath='{.items[].status.url}')/hello
```

## DEMO: Eventing system

### Deploy knative eventing

```yaml
# knative-eventing.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: knative-eventing
---
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeEventing
metadata:
  name: knative-eventing
  namespace: knative-eventing
```
Check:

```bash
oc get knativeeventing knative-eventing -n knative-eventing
```

### Deploy a default broker

```bash
oc label namespace demo eventing.knative.dev/injection=enabled
oc get broker default
```

### Remove a showcase ksvc template metadata name

**NOTE**: SinkBinding will not work if template metadata name is set on Ksvc ([knative/serving#9544](https://github.com/knative/serving/issues/9544)).

```bash
kn service update showcase --revision-name ''
```

### Add SinkBinding to Broker

```bash
kn source binding create showcase \
  --subject Service:serving.knative.dev/v1:showcase \
  --sink broker:default
```

### Deploy two event displays

```bash
kn service create event-display-1 \
  --image quay.io/openshift-knative/knative-eventing-sources-event-display:latest
kn service create event-display-2 \
  --image quay.io/openshift-knative/knative-eventing-sources-event-display:latest
```

### Add trigger

```bash
kn trigger create showcase-1 \
  --broker default \
  --sink ksvc:event-display-1
#  --filter <KEY=VALUE>
kn trigger create showcase-2 \
  --broker default \
  --sink ksvc:event-display-2
```
