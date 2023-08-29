# Red Hat Service Mesh Demo

This repository contains the required procedure to deploy an application in Openshift using service integration. The idea of this demo is show the Red Hat Service Mesh features managing traffic flow.

## Prerequisites

- A Red Hat Openshift Cluster 4.8+
- OC Client 4.8+ with cluster-admin role
- Web browser

## Setting Up

- Install Red Hat Service Mesh, Kiali and Red Hat OpenShift distributed tracing platform operators

```$bash
oc apply -f 00-service-mesh-operators.yaml
oc get po -n openshift-operators -w
```

- Deploy the Red Hat Service Mesh Control Plane - SMCP (*Please review the istiod, prometheus, grafana, jaeger and kiali pods are up and running*)

```$bash
oc apply -f 01-service-mesh-control-plane.yaml
oc get po -n istio-system -w
oc get smcp -n istio-system
```

- Define the Red Hat Service Mesh Member Roll (SMMR)

```$bash
oc apply -f 02-service-mesh-members-roll.yaml -n istio-system
```

- Deploy the application

```$bash
oc apply -f 03-jump-app-deploy.yaml
```

NOTE: It is required the correct Openshift applications domain is defined in every object properly 

- Test the application via Browser and Kiali (_Please find the URL executing the following command_)

```$bash
oc -n istio-system get route back-golang-istio-system -o jsonpath='{.spec.host}' && \
oc -n istio-system get route front-javascript-istio-system -o jsonpath='{.spec.host}' && \
oc -n istio-system get route kiali -o jsonpath='{.spec.host}'
```

## Deployment Strategies
### B/G

- Migrate from B deployment to G

```$bash
oc apply -n jump-app -f 04-jump-app-deploy-blue-green.yaml
```

### Canary

- Steps 1 (90% A / 10% B)

```$bash
oc apply -n jump-app -f 05-jump-app-deploy-canary-step1.yaml
```

- Step 2 (60% A / 40% B)

```$bash
oc apply -n jump-app -f 05-jump-app-deploy-canary-step2.yaml
```

- Step 3 (20% A / 80% B)

```$bash
oc apply -n jump-app -f 05-jump-app-deploy-canary-step3.yaml
```

- Step 3 (0% A / 100% B)

```$bash
oc apply -n jump-app -f 05-jump-app-deploy-canary-step4.yaml
```

### Mirror

- Mirror 100% traffic between versions in Python microservice

```$bash
oc apply -n jump-app -f 06-jump-app-deploy-mirror.yaml
```

## Chaos Monkey

### Delay

In this scenario, it is introduced a 3 seconds delay in every service Python request will be checked through the Kiali console.

```$bash
oc apply -n jump-app -f 07-jump-app-chaos-delay.yaml
```

### Abort
In this scenario, a crash failure is introduced in the back-python service. The fault injection error could be visualized in Kiali (FI flag).

```$bash
oc apply -n jump-app -f 07-jump-app-chaos-abort.yaml
```
### Timeout

In this scenario, it is introduced a fault injection to generate delays in back-springboot service requests and defined a connection timeout in the previous microservice, back-golang. The request fails due to the timeout.

Once the configuration is applied, it is possible to check the current state of the connection requests between services through the Kiali console.

```$bash
oc apply -n jump-app -f 08-jump-app-chaos-timeout.yaml
```

## Circuit Breaking & Outlier Detection

### Circuit Breaking

In this scenario, it is introduced a circuit breaking rule in order to limit the impact of failures and latency spikes. The idea is to configure the Python service in order to limit to a single connection and request to it.

```$bash
oc apply -n jump-app -f 09-jump-app-chaos-circuit-breaking-base.yaml
oc apply -n jump-app -f 09-jump-app-chaos-circuit-breaking.yaml
```

In Kiali, there is a 'ray' icon in the back-python application square that identify the presence of a CB definition.

Check that the jumpapp application is working properly:

```$bash
while true; do curl -k 'https://back-golang-istio-system.apps.${EXTERNAL_DOMAIN}/jump' --data-raw '{"message":"hello","jump_path":"/jump","last_path":"/jump","jumps":["http://back-springboot:8443","http://back-python:8444"]}' ; echo " "; sleep 0.5 ; done
```

Let’s now generate some load by adding a 10 clients calling out jumpapp app:

```$bash
seq 1 10 | xargs -n1 -P10 curl -k 'https://back-golang-istio-system.apps.${EXTERNAL_DOMAIN}/jump' --data-raw '{"message":"hello","jump_path":"/jump","last_path":"/jump","jumps":["http://back-springboot:8443","http://back-python:8444"]}'
```

### Outlier Detection

In this scenario, it is introduced a circuit breaking rule in order to avoid sending connections to an unavailable service. The idea is to configure the Python destination rule in order to be able to detect errors and avoid sending requests to this failed microservice.

Two replicas of each service will be deployed:

```$bash
oc scale -n jump-app deploy back-python-v1 --replicas=2
oc scale -n jump-app deploy back-python-v2 --replicas=2
```

Then, let’s randomly make one pod of our back-python service to fail by executing:

```$bash
oc exec -n jump-app $(oc get pods -o NAME | grep back-python-v1 | tail -n 1) -- curl -s localhost:8080/faulty -X POST
```

And run some tests now. Let’s have a look at the output as there will be some failures comming from an unknown (yet) back-python pod:

```$bash
while true; do date +%H:%M:%S ; curl -k 'https://back-golang-istio-system.apps.${EXTERNAL_DOMAIN}/jump' --data-raw '{"message":"hello","jump_path":"/jump","last_path":"/jump","jumps":["http://back-springboot:8443","http://back-python:8444"]}' ; echo " "; sleep 1; done
```

It is time to make our services mesh more resiliant and see the effect of applying an OutlierDetection policy over back-python service:

```$bash
oc apply -n jump-app -f 09-jump-app-chaos-circuit-breaking-base.yaml
oc apply -n jump-app -f 09-jump-app-chaos-outlier-detection.yaml
```

## Author

Asier Cidon @RedHat

Fran Perea @RedHat

## Edited by

Carlos Estay @RedHat
