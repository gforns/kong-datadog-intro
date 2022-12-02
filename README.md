# Monitor Kong using Datadog 

We can use Datadog to monitor Kong in a Kubernetes environment.
These tests are done using Minikube and everything is deploied in `default` namespace.

## Datadog Account preparation:

1. Create an account in Datadog. In this example, I will be using European site of Datadog: 
- https://datadoghq.eu

2. Create an `apiKey` on your account and save the key:
- https://app.datadoghq.eu/personal-settings/application-keys

3. Export the apiKey as a environmenet variable:
```
export DD_API_KEY=12345678901234567890
```


## Install Kong

Add Kong helm charts repo:
```
helm repo add kong https://charts.konghq.com
helm repo update
```

Install Kong using latest Kong 2.8 helm chart: (--versioni v2.12.0)
```
helm install kong kong/kong --version v2.12.0
```

And let's create a Deployment / Service / Ingress

```
kubectl apply -f httpbin-deployment-service.yaml
```


## Install Datadog Agent using helm chart


Add Datadog helm repo:
```
helm repo add datadog https://helm.datadoghq.com
helm repo update
```

Install Datadog Agent we need to specify the `datadog.site` and `datadog.apiKey`:
```
helm install datadog \
--set datadog.site=datadoghq.eu \
--set datadog.apiKey=$DD_API_KEY \
datadog/datadog
```


## Collect all K8s logs in Datadog:

Datadog can monitor all logs from the k8s cluster using the Autodiscovery.

We need to set to `true` the `datadog.logs.enabled` and  `datadog.logs.containerCollectAll`:

And if we are using `minikube` we need to disable `datadog.kubelet.tlsVerify`:

```
helm upgrade -i datadog  \
--set datadog.site=datadoghq.eu \
--set datadog.apiKey=$DD_API_KEY \
--set datadog.logs.enabled=true \
--set datadog.logs.containerCollectAll=true \
--set datadog.kubelet.tlsVerify=false \
datadog/datadog
```


## Collect only some logs in Datadog


Apply above configuration but keeping `datadog.logs.containerCollectAll`to `false`:


```
helm upgrade -i datadog  \
--set datadog.site=datadoghq.eu \
--set datadog.apiKey=$DD_API_KEY \
--set datadog.logs.enabled=true \
--set datadog.kubelet.tlsVerify=false \
datadog/datadog
```


Then we need to specify in each pod if we want to get the logs.

Create a `kong-values.yaml` file and add:

```
podAnnotations:
  ad.datadoghq.com/proxy.logs: '[{"source": "kong", "service": "kong-proxy"}]'
  ad.datadoghq.com/ingress-controller.logs: '[{"source": "kong", "service": "kong-ingress-controller"}]'
```

Apply the changes to Kong:

```
helm upgrade kong -f kong-values.yaml --version=v2.12.0 kong/kong
```


## Get metrics in Datadog using OpenMetrics (Prometheus)

Datadog has some Kong dasbhoards that are using OpenMetrics from Kong <= 2.8. 
In Kong Gateway 3.0.0 some names from Prometheus metrics have been modified:
- https://docs.konghq.com/hub/kong-inc/prometheus/#changelog


Apply the Prometehus 'KongClusterPlugin':

```
kubectl apply -f prometheus-plugin.yaml
```


In the `kong-values.yaml` file add:

```
podAnnotations:
  ad.datadoghq.com/proxy.check_names: '["kong"]'
  ad.datadoghq.com/proxy.init_configs: '[{}]'
  ad.datadoghq.com/proxy.instances: '[{"openmetrics_endpoint": "http://%%host%%:8100/metrics"}]'
```


Apply the changes to Kong:

```
helm upgrade kong -f kong-values.yaml --version=v2.12.0 kong/kong
```

Create a Sample service

```
kubectl apply -f httpbin-deployment-service.yaml
```

send request to the Ingress and you should see data in the Kong default dashboards.



## Get metrics in Datadog using Kong Plugin

If Kubernetes Autodiscovery is not enabled or working, Kong can directly send some metrics to Datadog Agent using the plugin:

```
kubectl apply -f datadog-plugin.yaml
```

In the config of the plugin there is `prefix: kongmyplugin`, so in Datadog you should see some new metrics starting with `kongmyplugin.*`


