# Step 1: Run & expose php-apache server 

To demonstrate Horizontal Pod Autoscaler we will use a custom docker image based on the php-apache image. 

The Dockerfile has the following content:

<img width="272" alt="CleanShot 2022-09-06 at 11 28 33@2x" src="https://user-images.githubusercontent.com/193215/188599696-de3dd9ba-ba41-491d-8f38-2853b9d0cf56.png">

It defines an index.php page which performs some CPU intensive computations:

<img width="389" alt="CleanShot 2022-09-06 at 11 28 55@2x" src="https://user-images.githubusercontent.com/193215/188599766-2f7f9ca1-595f-4a34-b947-861cc8449211.png">

First, we will start a deployment running the image and expose it as a service using the following configuration in php-apache.yaml

```kubectl apply -f https://k8s.io/examples/application/php-apache.yaml```

# Step 2: Create Horizontal Pod Autoscaler

Now that the server is running, we will create the autoscaler using kubectl autoscale. 
The following command will create a Horizontal Pod Autoscaler that maintains between 1 and 10 replicas of the Pods controlled by the php-apache deployment we created in the first step of these instructions. 
Roughly speaking, HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% (since each pod requests 200 milli-cores by kubectl run), this means average CPU usage of 100 milli-cores). 

```kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10```

We may check the current status of autoscaler by running:kubectlget hpaPlease note that the current CPU consumption is 0% as we are not sending any requests to the server (the TARGET column shows the average across all the pods controlled by the corresponding deployment).

# Step 3: Increase load

Now, we will see how the autoscaler reacts to increased load. 
We will start a container, and send an infinite loop of queriesto the php-apache service (please run it in a different terminal):

```kubectl run -it --rm load-generator --image=busybox /bin/sh```

Hit enter for command prompt 

```while true; do wget -q -O - http://php-apache; done```

Within a minute or so, we should see the higher CPU load by executing:

```kubectl get hpa```

Here, CPU consumption has increased to 305% of the request.
As a result, the deployment was resized to 7 replicas:

```kubectl get deployment php-apache```


# Step 4: Stop load

We will finish our example by stopping the user load.
In the terminal where we created the container with busybox image, terminate the load generation by typing "Ctrl+C".
Then we will verify the result state (after a minute or so):

```kubectl get hpa```

Cooldown of scaling events
As the horizontal pod autoscaler checks the Metrics API every 30 seconds, previous scale events may not have successfully completed before another check is made. This behavior could cause the horizontal pod autoscaler to change the number of replicas before the previous scale event could receive application workload and the resource demands to adjust accordingly.

To minimize race events, a delay value is set. This value defines how long the horizontal pod autoscaler must wait after a scale event before another scale event can be triggered. This behaviorallows the new replica count to take effect and the Metrics API to reflect the distributed workload. 
There is no delay for scale-up events as of Kubernetes 1.12, however the delay on scale down events is defaulted to 5 minutes. Currently, you can't tune these cooldownvalues from the default on AKS.

# Step 5: Custom Metrics

Now, to make the desired metric available to the Horizontal Pod Autoscaler, it must first be added to a Metrics Registry. Andlast but not least, a custom metrics API must provide access to the desired metric for the Horizontal Pod Autoscaler.

To install Prometheus, workshop chose “kube-prometheus” (https://github.com/coreos/kube-prometheus) which installs Prometheus as well as Grafana (and Alertmanager etc.) and is super easy to use! 
So first, clone the project to your local machine and deploy it to your cluster:

```git clone https://github.com/coreos/kube-prometheus```

from within the cloned repo...

```kubectl apply -f manifests/setup```

```kubectl apply -f manifests/```

# Step 6: Public access to Grafana and Prometheus

```kubectl edit svc service_name -n namespace```

```i - to edit the service```

```ESC, :wq - update your service```

Change type LoadBalancer
Remove status : loadbalancer: {}

If you are promted for a username and password, default is:

“admin”
”admin”

you need to change that on the first login:

# Step 7: Metrics Source / Sample Application

To demonstrate how to work with custom metrics, we will use a very simple NodeJS application that provides a single endpoint (from the application perspective). If requests are sent to this endpoint, a counter (Prometheus Gauge –later more on other options) is set with a value provided from the body of the request. 
The application itself uses the Express framework and an additional library that allows to “interact” with Prometheus (prom-client) –to provide metrics via a /metrics endpoint.

This is how it looks:

<img width="376" alt="CleanShot 2022-09-06 at 11 52 49@2x" src="https://user-images.githubusercontent.com/193215/188605078-f6955c82-0ed9-4894-a5a3-226c9de1f0ab.png">

As you can see in the sourcecode, a “Gauge” metric is created – which is one of the types, Prometheus supports. 
Here’s a list of what metrics are offered (description from the official documentation):

Counter – a cumulative metric that represents a single monotonically increasing counter whose value can only increase or be reset to zero on restart

Gauge – a gauge is a metric that represents a single numerical value that can arbitrarily go up and down

Histogram – a histogram samples observations (usually things like request durations or response sizes) and counts them in configurable buckets. It also provides a sum of all observed values.

Summary – Similar to a histogram, a summary samples observations (usually things like request durations and response sizes). While it also provides a total count of observations and a sum of all observed values, it calculates configurable quantiles over a sliding time window.

Let’s deploy the application (plus a service for it) with the following YAML manifest:

```kubectl apply -f prom-deployment.yaml```

Now that we have Prometheus installed and an application that exposes a custom metric, we also need to tell Prometheus to scrape the /metrics endpoint (BTW, this endpoint is automatically created by one of the libraries used in the app). 
Therefore, we need to create a ServiceMonitor which is a custom resource definition from Prometheus, pointing “a source of metrics”.

```kubectl apply -f prom-deployment.yaml```

# Step 8: ServiceMonitor 

What that basically does, is telling Prometheus to look for a service called “promtest” and scrape the metrics via the (default) endpoint /metrics on the http port (which is set to port 4000 in the Kubernetes service).The /metricsendpoint reports values like that:

```curl --location --request GET 'http://<EXTERNAL_IP_OF_SERVICE>:4000/metrics' | grep custom_metric```

# Step 9: Validate metric from Prometheus

When the ServiceMonitor has been applied, Prometheus will be able to discover the pods/endpoints behind the service and pull the corresponding metrics. In the web UI, you should be able to see the following “scrape target” Check prometheustargets and validate custom metric:

```http://<EXTERNAL_IP_OF_PROMETHEUS_SERVICE>:9090/targets```

# Step 10: Validate Grafana

Time to test the environment! First, let’s see how the current metric looks like. Open Grafana an have a look at the Custom dashboard (you can find the JSON for the dashboard in the GitHub repo mentioned at the end of the post). You see, we have one pod running in the cluster reporting a value of “1” at the moment 

Custom Dashboard:

grafana-dashboard.json

# Step 11: Validate api count

If everything is set up correctly, we should be able to call our service on /api/count and set the custom metric via a POST request with a JSON document that looks like that:

```curl --location --request POST 'http://<EXTERNAL_IP_OF_SERVICE>:4000/api/count' \--header 'Content-Type:application/json' \--data-raw '{"count": 7}'```

After setting the value via a POST request to “7”, Prometheus receives the updated value by scraping the metrics endpoint and Grafana isable to show the updated chart. To be able to execute the full example in the end on the basis of a “clean environment”, set the counter back to “1”.

# Step 12: Prometheus Adapter

So, now the adapter must be installed. Workshop will use the following implementation for this sample:

https://github.com/DirectXMan12/k8s-prometheus-adapter

The installation works quite smoothly, but you have to adapt a few small things for it.
First you should clone the repository 
```git clone https://github.com/DirectXMan12/k8s-prometheus-adapter```

Change the URL for the Prometheus server in the file 

k8s-prometheus-adapter/deploy/manifests/custom-metrics-apiserver-deployment.yaml. 

In the current case, this is 

```http://<EXTERNAL_IP_OF_PROMETHEUS_SERVICE>:9090/```

Furthermore, an additional rule for our custom metric must be defined in the ConfigMap, which defines the rules for mapping from Prometheus metrics to the Metrics API schema (k8s-prometheus-adapter/deploy/manifests/custom-metrics-config-map.yaml). 

Overwrite the rule with the custom metric:

custom-metrics-config-map.yaml

In our case, only the query for custom_metric_counter_total_by_podis executed and the results are mapped to the metrics schema astotal/sumvalues.

# Step 13: Create SSL Certificate

To enable the adapter to function as a custom metrics API in the cluster, a SSL certificate must be created that can be used by the Prometheus adapter. All traffic from the Kubernetes control plane components to each other must be secured by SSL. This SSL certificate must be addedupfront as a secret in the cluster, so that the Custom Metrics Server can automatically map it as a volume during deployment.
Clone cfssl under the k8s-prometheus-adapter folder.

```git clone https://github.com/cloudflare/cfssl.git```
```cd cfssl```
```make```

# Step 14: Run Certificate Generation

Run generation script
```bash ./gencerts.sh```

# Step 15: Create metric server

```kubectl create namespace custom-metrics```

```kubectl apply -f cm-adapter-serving-certs.yaml -n custom-metrics```

```kubectl apply -f deploy/manifests/```



# Step 16: Horizontal Pod Scaler

Now we need to deploy a Horizontal Pod Autoscaler that targets our app deployment and references the custom metric 

code custom-metric-hpa.yaml

```kubectl apply -f custom-metric-hpa.yaml```


# Step 17: Trigger AutoScale

Now that we have added the HPA manifest, we can make a new request against our API to increase the value of the counter back to “7”. Please keep in mind that the target value for the HPA was “3”. This means that the Horizotal Pod Autoscaler should scale our deployment to a total of three pods after a short amount of time. 
Let’s see what happens:

```curl --location --request POST'http://<EXTERNAL_IP_OF_SERVICE>:4000/api/count' --header 'Content-Type: application/json' --data-raw '{ "count": 7 }'```

And last but not least, the Horizontal Pod Autoscaler does its job and scales the deployment the three pods
Validate through Grafana

```kubectl get events```


