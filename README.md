## Kubernetes APIs for monitoring & observability

This repository contains examples as shared in a July 2020 [CNCF webinar][0] I 
co-host with CNCF Ambassador [Chris Short][chrisshort], entitled "The top 7 
most useful Kubernetes APIs for comprehensive cloud native observability". 

Related: 

- Abstract & slides: [CNCF Member Webinar: The top 7 most useful Kubernetes 
  APIs for comprehensive cloud native observability][0] 
- Video (YouTube): [Webinar: The top 7 most useful Kubernetes APIs for comprehensive
  cloud native observability][1]
- Discussion (Discourse): [Webinar followup: Caleb’s CNCF webinar on K8s APIs 
  for observability][2]
  
## Setup

For the purposes of these examples, we will be accessing the Kubernetes APIs 
via [`kube-proxy`][3]. The following examples were written with for linux or 
MacOS users, so Windows users may need to 

> NOTE: This should be obvious to anyone reading these instructions, but just
> to be safe, please note the following prerequisites for running through 
> these examples: 
> 
> - A running Kubernetes cluster
> - The `kubectl` CLI utility 
> - `curl` (or some familiarity with translating commong curl commands to 
>   Powershell/etc)
> - [`jq`][4] (mostly for convenience & readability of JSON output)
> 
> Installing these components is left as an exercise for the reader (or in the 
> case of Kubernetes – your favorite ☁️ provider). 😅

1. Start the `kubectl` proxy. 

   ```shell
   $ kubectl proxy --port 8001
   ```
   
   > _NOTE: the default proxy port is 8001, so it's not necessary to provide 
   > the `--port` argument; I'm simply showing it here for convenience. If you
   > already have something listening on port 8001, you'll need to pick a 
   > different port._

2. Set some environment variables. 

   For convenience only – let's set some environment variables in case we 
   need to work from a different namespace than `default`. 

   ```shell
   $ export K8S_NAMESPACE=default
   ```
   
3. Verify the proxy connection. 

   ```shell
   $ curl -I "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods"  
   ```
   
   > _NOTE: you will need to run this and all subsequent `curl` and `kubectl` 
   > commands in adifferent terminal window or pane._

   
## Examples 

If you've started using Kubernetes you've likely already been introduced to the 
superb `kubectl` CLI (the inspiration for Sensu's own [`sensuctl`][6] CLI). 
   
The following examples are intended to help new Kubernetes users become more 
familiar with the Kubernetes APIs. To that end, most of the examples are simply
reproducing access to data that is already available via `kubectl` (which is 
just an API client). I hope this is helpful! 

Most Kubernetes APIs offer methods to "list" (get multiple resources), "read" 
(get a single resource), and "write" (create or update a resource). I won't 
show examples for every method on every endpoint, but I encourage you to 
read the API documentation (especially if you're just learning how to read API
docs) and experiment! This should be fun! 

OK, here we go!

### Service API 

A [Kubernetes Service][5] is "an abstract way to expose an application running 
on a set of Pods as a network service". The [Kubernetes Networking Concepts][5]
documentation provides a great introduction to Services. 

1. Listing services 
   
   **`kubectl` CLI**
   
   ```shell
   $ kubectl get services --namespace ${K8S_NAMESPACE}
   ```
   
   **Kubernetes API**

   ```shell
   $ curl -X GET "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/services"
   ```
   
2. Get information about a specific service 

   **`kubectl` CLI**
   
   ```shell
   $ export K8S_SERVICE=example
   $ kubectl inspect service ${K8S_SERVICE} --namespace ${K8S_NAMESPACE}
   NAME     TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)           AGE
   nginx    LoadBalancer   XX.XX.XX.XX     XX.XX.XX.XX    8888:30275/TCP    137d
   ```
   
   **Kubernetes API**
   
   ```shell
   $ curl -X GET -s "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/services/${K8S_SERVICE}"
   {
     "kind": "Service",
     "apiVersion": "v1",
     "metadata": {
       "name": "nginx",
       "namespace": "default",
       "selfLink": "/api/v1/namespaces/default/services/nginx",
       "uid": "f2655d2d-6fa4-11ea-9f15-42010a8a0079",
       "resourceVersion": "116850677",
       "creationTimestamp": "2020-03-26T21:01:22Z",
       "annotations": {
         "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"name\":\"nginx\",\"namespace\":\"default\"},\"spec\":{\"ports\":[{\"name\":\"http\",\"port\":8888,\"protocol\":\"TCP\",\"targetPort\":80}],\"selector\":{\"app\":\"nginx\"},\"type\":\"LoadBalancer\"}}\n"
       }
     },
     "spec": {
       "ports": [
         {
           "name": "http",
           "protocol": "TCP",
           "port": 8888,
           "targetPort": 80,
           "nodePort": 30275
         }
       ],
       "selector": {
         "app": "nginx"
       },
       "clusterIP": "XX.XX.XX.XX",
       "type": "LoadBalancer",
       "sessionAffinity": "None",
       "externalTrafficPolicy": "Cluster"
     },
     "status": {
       "loadBalancer": {
         "ingress": [
           {
             "ip": "XX.XX.XX.XX"
           }
         ]
       }
     }
   }
   ```
   
### Pod API 

1. 

### Downward API

```shell
$ sensuctl entity list --namespace cncf-webinar --format json | jq '.[] | { 
    name: .metadata.name, 
    namespace: .metadata.namespace, 
    labels: .metadata.labels
  }'
```

### Events API

1. Get events from the API 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8888/api/v1/namespaces/cncf-webinar/events" | less
   ```

   Alternatively, to see more readable output, get an EventList with a jq filter: 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8888/api/v1/namespaces/cncf-webinar/events?fieldSelector=type%21%3DNormal" | jq '.items[] | {
       type: .type,
       message: .message,
       reason: .reason,
       kind: .involvedObject.kind,
       name: .involvedObject.name
     }'
   ```


### Kubernetes API Watchers 

1. Start the `kubectl` proxy. 

   ```shell
   $ kubectl proxy --port 8888
   ```

2. In a separate terminal/pane, get a PodList 

   ```shell
   $ kubectl get pods
   $ curl -XGET -s "http://127.0.0.1:8888/api/v1/namespaces/cncf-webinar/pods" | less 
   ```

   Notice the response type of `PodList` – this is a Kubernetes API resource 
   _collection_. This `PodList` resource has a `resourceVersion`. 

3. Copy the PodList `resourceVersion` 

   ```shell
   $ PODLIST_VERSION=$(curl -XGET -s "http://127.0.0.1:8888/api/v1/namespaces/cncf-webinar/pods" | jq -r .metadata.resourceVersion)
   ```

4. Start a PodList watcher 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8888/api/v1/watch/namespaces/cncf-webinar/pods?resourceVersion=${PODLIST_VERSION}" 
   ```

   Alternatively, to see more readable output, start a PodList watcher with a jq filter: 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8888/api/v1/watch/namespaces/cncf-webinar/pods?resourceVersion=${PODLIST_VERSION}" | jq '. | {
       action: .type,
       type: .object.kind,
       name: .object.metadata.name
     }'
   ```

5. In a separate terminal/pane, modify the PodList by adding or deleting a pod

   ```shell
   $ kubectl scale deployment nginx --replicas 3
   ```




[chrisshort]: https://github.com/chris-short 
[0]: https://www.cncf.io/webinars/the-top-7-most-useful-kubernetes-apis-for-comprehensive-cloud-native-observability/ 
[1]: https://youtu.be/5LjTApwydao 
[2]: https://discourse.sensu.io/t/webinar-followup-calebs-cncf-webinar-on-k8s-apis-for-observability/1945 
[3]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
[4]: https://stedolan.github.io/jq/ 
[5]: https://kubernetes.io/docs/concepts/services-networking/service/ 
[6]: https://docs.sensu.io/sensu-go/latest/sensuctl/ 


