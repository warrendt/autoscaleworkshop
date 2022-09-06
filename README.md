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

```while true; do wget -q -O-http://php-apache; done```

Within a minute or so, we should see the higher CPU load by executing:

```kubectl get hpa```

Here, CPU consumption has increased to 305% of the request.
As a result, the deployment was resized to 7 replicas:

```kubectl get deployment php-apache```


# Step 4: Stop load

We will finish our example by stopping the user load.
In the terminal where we created the container with busybox image, terminate the load generation by typing "Ctrl+C".
Then we will verify the result state (after a minute or so):

```kubectlget hpa```

Cooldown of scaling events
As the horizontal pod autoscaler checks the Metrics API every 30 seconds, previous scale events may not have successfully completed before another check is made. This behavior could cause the horizontal pod autoscaler to change the number of replicas before the previous scale event could receive application workload and the resource demands to adjust accordingly.

To minimize race events, a delay value is set. This value defines how long the horizontal pod autoscaler must wait after a scale event before another scale event can be triggered. This behaviorallows the new replica count to take effect and the Metrics API to reflect the distributed workload. 
There is no delay for scale-up events as of Kubernetes 1.12, however the delay on scale down events is defaulted to 5 minutes. Currently, you can't tune these cooldownvalues from the default on AKS.

# Step 5: Custom Metrics

Now, to make the desired metric available to the Horizontal Pod Autoscaler, it must first be added to a Metrics Registry. Andlast but not least, a custom metrics API must provide access to the desired metric for the Horizontal Pod Autoscaler.

To install Prometheus, workshop chose “kube-prometheus” (https://github.com/coreos/kube-prometheus) which installs Prometheus as well as Grafana (and Alertmanager etc.) and is super easy to use! 
So first, clone the project to your local machine and deploy it to your cluster:git clone https://github.com/coreos/kube-prometheus# from within the cloned repo...

```kubectl apply -f manifests/setup```
```kubectl apply -f manifests/```

# Step 6: Public access to Grafana and Prometheus

```kubectl edit svc service_name -n namespace```
```i -to edit the service```
```ESC, :wq -update your service```

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

```kubectl apply -f prom-deploy.yaml```



