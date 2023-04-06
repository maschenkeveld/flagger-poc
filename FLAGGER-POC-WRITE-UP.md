# Flagger POC Write-Up

## Prerequisites

- Metrics enabled in the mesh definitions
- A "local" Prometheus is running in the mesh, that is setup to remote write to the "main" Prometheus.
- Main Prometheus reachable from Flagger

## Flagger Installation

Flagger can be installed using Helm using the Flux/Kustomize configuration attached. Make sure to update the `metricsServer` field. If the metrics server uses a private CA, this needs to be added to the (self hosted) Flagger docker image.

The Mesh provider is set to Kuma: `meshProvider: "kuma"` and the for multi-zone support the controlplane has te be configured as follows:

```
kubeconfig:
  # controlplane.kubeconfig.secretName: The name of the secret containing the mesh control plane kubeconfig
  secretName: "gcp-kubeconfig"
  # controlplane.kubeconfig.key: The name of secret data key that contains the mesh control plane kubeconfig
  key: "kubeconfig"
```

This will the secret `gcp-kubeconfig` to be present in the namespace, see my Flux/Kustomize configuration for an example. This Kube config file can be found on the Global Control Plane cluster.

## Application / Microservice / Spring Boot installation

Deploy the (Spring Boot) application into a namespace that is mesh annotated. But do **not deploy** a service.

Use the following Canary configuration to let Flagger create the required Kubernetes services. These services will get picked up by Kong Mesh, which will create mesh services for them.

```
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: alice
  namespace: apps-a-mesh-a
  annotations:
    kuma.io/mesh: mesh-a
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: alice
  service:
    # service name (optional)
    name: alice
    # ClusterIP port number (required)
    port: 8080
    # container port name or number
    targetPort: http
    # port name can be http or grpc (default http)
    portName: http
    apex:
      annotations:
        8080.service.kuma.io/protocol: "http"
    canary:
      annotations:
        8080.service.kuma.io/protocol: "http"
    primary:
      annotations:
        8080.service.kuma.io/protocol: "http"
  analysis:
    # schedule interval (default 60s)
    interval: 30s
    # max number of failed metric checks before rollback
    threshold: 5
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    metrics:
      - name: request-success-rate
        threshold: 99
        interval: 1m
      - name: request-duration
        threshold: 500
        interval: 30s
```

## Test the deployment and service

Make sure the application is reachable over the Kong Mesh service, for example via a Kong Gateway dataplane that runs in the same mesh.

## Run a load test on the application

Run a load on the application, this can be the same request you used for testing the deployment and service. K6 (https://k6.io/) is a good tool for this, see a sample configuration attached. 

After changing the `baseURL` and `path`, you can run this by executing `k6 run single.js`

## Prepare for a Canary deployment

Make a change to the application deployment, for example update the Docker image version it is using, this will resemble a real world scenario.

Do the deployment using Flux.

## Flagger will now do the rest

- It will attach a Kong Mesh `application-canary` service to the new deployment
- It will create a Traffic Route policy on the Global Control Plane that splits the traffic to `application` between `application-primary` and `application-canary` (95%, 5%)
- It will look at the main prometheus and look for envoy metrics on HTTP response codes and request latency for the new deployment
- If the metrics stay withing the defined threshold, it will dynamicly update the Traffic Route
- If the traffic remains stable during a 50% 50% split, it will
    - Redeploy the "canary" pod as the `application-primary` pod
    - Set the Traffic Route to route 100% of the traffic to this primary pod
    - Decomission the old deployment
    - Wait for the next update to the deployment

## Metrics

This is what Flagger is looking at on the main Prometheus server:

**request-success-rate**

```
sum(
	rate(
		envoy_cluster_upstream_rq{
			envoy_cluster_name=~"{{ target }}-canary_{{ namespace }}_svc_[0-9a-zA-Z-]+",
			envoy_response_code!~"5.*"
		}[{{ interval }}]
	)
) 
/ 
sum(
	rate(
		envoy_cluster_upstream_rq{
			envoy_cluster_name=~"{{ target }}-canary_{{ namespace }}_svc_[0-9a-zA-Z-]+",
		}[{{ interval }}]
	)
) 
* 100
```

**request-duration**

```
histogram_quantile(
	0.99,
	sum(
		rate(
			envoy_cluster_upstream_rq_time_bucket{
				envoy_cluster_name=~"{{ target }}-canary_{{ namespace }}_svc_[0-9a-zA-Z-]+",
			}[{{ interval }}]
		)
	) by (le)
)
```

https://andykuszyk.github.io/2020-07-24-prometheus-histograms.html