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
MacOS users, so Windows users may need to take additional configuration steps
or use alternative commands not documented here.

> NOTE: This should be obvious to anyone reading these instructions, but just
> to be safe, please note the following prerequisites for running through 
> these examples: 
> 
> - A running Kubernetes cluster
> - The `kubectl` CLI utility 
> - `curl` (or some familiarity with translating common curl commands to 
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

   > _NOTE: you will need to run all subsequent `curl` and `kubectl` 
   > commands in a different terminal window or pane._

2. Set some environment variables. 

   For convenience only – let's set some environment variables in case we 
   need to work from a different namespace than `default`. 

   ```shell
   $ export K8S_NAMESPACE=default
   ```

   > _NOTE: you will want to make sure this variable is exported in all terminals 
   >  where you plan to run subsequent `curl` and `kubectl` commands._
   
3. Verify the proxy connection. 

   ```shell
   $ curl "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods"  
   ```

4. Setup the basic example applications.

   To make things easier for you, I've predefined a simple nginx deployment in this repository. The deployment sets up running nginx pods using a deployment controller, and exposes nginx as a load balanced service.

   ```shell
   $ kubectl apply -f kubernetes/nginx.yaml -n default
   $ kubectl create namespace sensu-system
   $ kubectl apply -f kubernetes/sensu-backend.yaml -n sensu-system
   ```

   > _NOTE: if you are using minikube locally instead of a cloud provider, you made 
   >  need to also apply the pv.yaml file before loading the sensu-backend.yaml
   >  to define a persistent volume manually.

   > _NOTE: if you are using minikube locally instead of a cloud provider, you made 
   >  need to also run minikube tunnel or use minikube service to expose the 
   > running load balanced services outside of the k8s cluster._

5. Check to makesure example applications are running

   Before moving on and working with the API examples, it's good to make sure the applications are ready to go.  
 
   ```shell
   $ kubectl get deployments -n default
     NAME    READY   UP-TO-DATE   AVAILABLE   AGE
     nginx   3/3     3            3           3m
    
   $ kubectl get statefulsets -n sensu-system
     NAME            READY   AGE
     sensu-backend   3/3     3m
   ```
   The important thing here is to make sure the application pods are all READY.  If they are not, you may need to take some extra steps to configure your k8s cluster. 

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
   NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
   kubernetes   ClusterIP      X.X.X.X         <none>          443/TCP          8m53s
   nginx        LoadBalancer   X.X.X.X         X.X.X.X         8888:30645/TCP   2m2s
   ```
   
   **Kubernetes API**

   ```shell
   $ curl -X GET "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/services"
   ```
   
2. Get information about a specific service 

   **`kubectl` CLI**
   
   ```shell
   $ export K8S_SERVICE=nginx
   $ kubectl describe service ${K8S_SERVICE} --namespace ${K8S_NAMESPACE}
   Name:                     nginx
   Namespace:                default
   Labels:                   <none>
   Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"ports":[{"name":"http","po...
   Selector:                 app=nginx
   Type:                     LoadBalancer
   IP:                       X.X.X.X
   LoadBalancer Ingress:     X.X.X.X
   Port:                     http  8888/TCP
   TargetPort:               80/TCP
   NodePort:                 http  30645/TCP
   Endpoints:                
   Session Affinity:         None
   External Traffic Policy:  Cluster
   Events:                   <none>

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

1. Listing Pods

   **`kubectl` CLI**

   ```shell
   $ kubectl get pods --namespace ${K8S_NAMESPACE}
     NAME                     READY   STATUS    RESTARTS   AGE
     nginx-6dbd474499-cjn95   1/1     Running   0          10s
     nginx-6dbd474499-qz9fx   1/1     Running   0          6s
   ```

   **Kubernetes API**

   ```shell
   $ curl -X GET "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods" | jq '.items[].metadata.name '
   "nginx-6dbd474499-cjn95"
   "nginx-6dbd474499-qz9fx"
   ```

2. Getting info for specific pod

   **`kubectl` CLI (abbreviated output)**
   ```shell
   $ kubectl describe pod nginx-6dbd474499-cjn95 --namespace ${K8S_NAMESPACE}
   Name:         nginx-6dbd474499-cjn95
   Namespace:    default
   Priority:     0
   Node:         minikube/192.168.99.106
   Start Time:   Thu, 27 Aug 2020 17:08:37 -0800
   Labels:       app=nginx
                 pod-template-hash=6dbd474499
   Annotations:  <none>
   Status:       Running

   ...

   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 

   ...

   Events:
     Type     Reason          Age                    From               Message
     ----     ------          ----                   ----               -------
     Normal   Scheduled       29m                    default-scheduler  Successfully assigned default/nginx-6dbd474499-cjn95 to minikube
     Normal   Pulling         29m                    kubelet, minikube  Pulling image "nginx:latest"
   ...

   ```
  
   **Kubernetes API (abbreviated output)**
   ```shell
   $ curl -X GET "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods/nginx-6dbd474499-cjn95" | jq '.metadata.name, .metadata.labels'
   nginx-6dbd474499-cjn95
   {
   "app": "nginx",
   "pod-template-hash": "6dbd474499"
   }
   ```
### Downward API
To explore the Downward API, you'll need to configure the Sensu commandline tool to communicate with the sensu-backend running in your k8s cluster.  If running in a public cloud environment, you can obtain the correct IP address and port to use for the Sensu backend service with:
   ```shell
   $ kubectl get service sensu-lb -n sensu-system
   ```
The sensu-backend api url will be `http://EXTERNAL-IP:8080`


  _NOTE: if you are using minikube locally instead of a cloud provider, you made 
  need to use `minikube service list -n sensu-system` to expose the 
  local network URL available outside of the k8s cluster._

   ```shell
   $ minikube service list -n sensu-system
   ```

The sensu-backend statefulset that I had you deploy earlier, has a sensu-agent sidecard running in each pod.  The Downward API is used to populated environment variables in the sensu-agent running environment with information concerning details of the k8s configuration. These envvar are then used in the configuration of the sensu-agent labels.

   ```shell
   $ sensuctl entity list --namespace default --format json | jq '.[] | { 
       name: .metadata.name, 
       labels: .metadata.labels
     }'

   {
     "name": "sensu-backend-0",
     "labels": {
       "foo": "bar",
       "kube_namespace": "sensu-system",
       "kubelet": "minikube"
     }
   }
   {
     "name": "sensu-backend-1",
     "labels": {
       "foo": "bar",
       "kube_namespace": "sensu-system",
       "kubelet": "minikube"
     }
   }
   {
     "name": "sensu-backend-2",
     "labels": {
       "foo": "bar",
       "kube_namespace": "sensu-system",
       "kubelet": "minikube"
     }
   }

   ```

### Events API

1. Get events from the API 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/events" | less
   ```

   Alternatively, to see more readable output, get an EventList with a jq filter: 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/events?fieldSelector=type%21%3DNormal" | jq '.items[] | {
       type: .type,
       message: .message,
       reason: .reason,
       kind: .involvedObject.kind,
       name: .involvedObject.name
     }'
   ```


### Kubernetes API Watchers 

1. In a separate terminal/pane, get a PodList 

   ```shell
   $ kubectl get pods
   $ curl -XGET -s "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods" | head -n 20
   ```

   Notice the response type of `PodList` – this is a Kubernetes API resource 
   _collection_. This `PodList` resource has a `resourceVersion`. 

3. Copy the PodList `resourceVersion` 

   ```shell
   $ PODLIST_VERSION=$(curl -XGET -s "http://127.0.0.1:8001/api/v1/namespaces/${K8S_NAMESPACE}/pods" | jq -r .metadata.resourceVersion)
   ```

4. Start a PodList watcher 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8001/api/v1/watch/namespaces/${K8S_NAMESPACE}/pods?resourceVersion=${PODLIST_VERSION}" 
   ```

   Alternatively, to see more readable output, start a PodList watcher with a jq filter: 

   ```shell
   $ curl -XGET -s "http://127.0.0.1:8001/api/v1/watch/namespaces/${K8S_NAMESPACE}/pods?resourceVersion=${PODLIST_VERSION}" | jq '. | {
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


