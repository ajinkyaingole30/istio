** Performed on “https://www.katacoda.com/courses/istio/deploy-istio-on-kubernetes”

###Istio service mesh components


Istio isa service mesh — an application-aware infrastructure layer for facilitating service-to-service communications. By ‘application-aware’, it is meant that the service mesh understands, to some degree, the nature of service communications and can intervene in a value-added manner. For example, a service mesh can implement resiliency patterns (retries, circuit breakers), alter the traffic flow (shape the traffic, affect routing behavior, facilitate canary releases), as well as add a whole host of comprehensive security controls. Being intrinsically aware of the traffic passing between services, Istio can also provide fine-grained instrumentation and telemetry insights, providing a degree of observability to an otherwise opaque distributed system.

Istio is backed by Google, IBM, and Lyft, and is currently the most widely-adopted service mesh architecture. Kubernetes, which was originally designed by Google, also dovetails nicely into Istio. It would be fair to label Istio as a ‘Kubernetes-native service mesh’.

##How it works




Like most service mesh implementations, Istio complements existing application containers with a proxy container, called a sidecar. Sidecar proxies are specially configured Envoy instances that intercept network traffic entering and leaving service containers and reroute the traffic over a dedicated network, as illustrated below.

 

Sidecar proxies are lightweight components optimized for latency and throughput, housing minimal configuration and routing intelligence. Routing decisions are made on the basis of policies hosted by a separate
control plane — the metaphorical ‘brain’ of the service mesh. The control plane comprises a dedicated set of components deployed into the Kubernetes cluster — much like any other containerized application — residing in a dedicated istio-system namespace.
data plane — the elements that perform the actual routing of network traffic and interface directly with the application containers. This separation is illustrated in the diagram below.
 




Core concepts
Istio expands upon the nomenclature of a ‘vanilla’ Kubernetes setup with several Istio-specific resource types. Being a Kubernetes-native service mesh, Istio designs have implemented these concepts using custom resource definitions (CRDs). CRDs are nothing more than declarative snippets of configuration specified in YAML and managed using kubectl, akin to the built-in types such as Pod, Service, Deployment, and so on.





How To Perform:

Deploy Istio
Istio is installed in two parts :-
The first part involves the CLI tooling that will be used to deploy and manage Istio backed services. The second part configures the Kubernetes cluster to support Istio.




Lets Deploy ISTIO:

$ curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.0 sh -

The following command will install the Istio 1.0.0 release.


After it has successfully run, add the bin folder to your path.

$ export PATH="$PATH:/root/istio-1.0.0/bin"


$ cd /root/istio-1.0.0

Configure Istio CRD( Custom Resource Definitions)

Custom resources. A resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind; for example, the built-in pods resource contains a collection of Pod objects. ... It represents a customization of a particular Kubernetes installation.
Istio has extended Kubernetes via Custom Resource Definitions (CRD). Deploy the extensions by applying crds.yaml.

$kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system


Install Istio with default mutual TLS authentication

TLS(Transport Layer Security ): Manage TLS Certificates in a Cluster. Kubernetes provides a certificates.k8s.io API, which lets you provision TLS certificates signed by a Certificate Authority (CA) that you control. These CA and certificates can be used by your workloads to establish trust.
To Install Istio and enforce mutual TLS authentication by default, use the yaml istio-demo-auth.yaml:

$kubectl apply -f install/kubernetes/istio-demo-auth.yaml

This will deploy Pilot, Mixer, Ingress-Controller, and Egress-Controller, and the Istio CA (Certificate Authority). These are explained in the next step.

Check status:
All the services are deployed as Pods.

Kubectl get pods -n istio-system 



Deploy Katacoda Service

To make the sample BookInfo application and dashboards available to the outside world, in particular, on Katacoda, deploy the following Yaml.

$kubectl apply -f /root/katacoda.yaml



Istio Architecture


Istio intro

The previous step deployed the Istio Pilot, Mixer, Ingress-Controller, and Egress-Controller, and the Istio CA (Certificate Authority).

Pilot - Responsible for configuring the Envoy and Mixer at runtime.
Proxy / Envoy - Sidecar proxies per microservice to handle ingress/egress traffic between services in the cluster and from a service to external services. The proxies form a secure microservice mesh providing a rich set of functions like discovery, rich layer-7 routing, circuit breakers, policy enforcement and telemetry recording/reporting functions.

Mixer - Create a portability layer on top of infrastructure backends. Enforce policies such as ACLs, rate limits, quotas, authentication, request tracing and telemetry collection at an infrastructure level.

Citadel / Istio CA - Secures service to service communication over TLS. Providing a key management system to automate key and certificate generation, distribution, rotation, and revocation.
Ingress/Egress - Configure path based routing for inbound and outbound external traffic.
Control Plane API - Underlying Orchestrator such as Kubernetes or Hashicorp Nomad.
The overall architecture is shown below.

Control Plane API - Underlying Orchestrator such as Kubernetes or Hashicorp Nomad.

The overall architecture is shown below.
 


Check Status :

$ kubectl get pods -n istio-system


Wait until they are all running or have completed. Once they're running, Istio has correctly been deployed.

Deploy Sample Application

To showcase Istio, a BookInfo web application has been created. This sample deploys a simple application composed of four separate microservices which will be used to demonstrate various features of the Istio service mesh.

When deploying an application that will be extended via Istio, the Kubernetes YAML definitions are extended via kube-inject. This will configure the services proxy sidecar (Envoy), Mixers, Certificates and Init Containers.


$ kubectl apply -f <(istioctlkube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)



Deploy gateway

$kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml


Check Status

$ kubectl get pods
 
When the Pods are starting, you may see initiation steps happening as the container is created. This is configuring the Envoy sidecar for handling the traffic management and authentication for the application within the Istio service mesh.
Once running the application can be accessed via the path /productpage.
https://2886795278-80-cykoria08.environments.katacoda.com/productpage






Apply default destination rules


Before you can use Istio to control the Bookinfo version routing, you need to define the available versions, called subsets, in destination rules.


$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml


Control Routing

One of the main features of Istio is its traffic management. As a Microservice architectures scale, there is a requirement for more advanced service-to-service communication control.




User Based Testing / Request Routing.

One aspect of traffic management is controlling traffic routing based on the HTTP request, such as user agent strings, IP address or cookies.

Similarly to deploying Kubernetes configuration, routing rules can be applied using istioctl.


$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml



Traffic Shaping for Canary Releases

The ability to split traffic for testing and rolling out changes is important. This allows for A/B variation testing or deploying canary releases.

The rule below ensures that 50% of the traffic goes to reviews:v1 (no stars), or reviews:v3 (red stars).


$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml



New Releases

Given the above approach, if the canary release were successful then we'd want to move 100% of the traffic to reviews:v3.

This can be done by updating the route with new weighting and rules.


$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml



List All Routes

It's possible to get a list of all the rules applied using


$ istioctl get virtualservices

$ istioctl get virtualservices -o yaml


Access Metrics

With Istio's insight into how applications communicate, it can generate profound insights into how applications are working and performance metrics.

Generate Load

To view the graphs, there first needs to be some traffic. Execute the command below to send requests to the application.


while true; do   curl -s https://2886795324-80-cykoria08.environments.katacoda.com/productpage > /dev/null;   echo -n .;   sleep 0.2; done




Access Dashboards

With the application responding to traffic the graphs will start highlighting what's happening under the covers.

Grafana

 
the first is the Istio Grafana Dashboard. The dashboard returns the total number of requests currently being processed, along with the number of errors and the response time of each call.

https://2886795278-3000-cykoria08.environments.katacoda.com/dashboard/db/istio-mesh-dashboard



As Istio is managing the entire service-to-service communicate, the dashboard will highlight the aggregated totals and the breakdown on an individual service level.

Jaeger

Jaeger provides tracing information for each HTTP request. It shows which calls are made and where the time was spent within each request.

https://2886795278-16686-cykoria08.environments.katacoda.com/





Click on a span to view the details on an individual request and the HTTP calls made. This is an excellent way to identify issues and potential performance bottlenecks.




Service Graph
As a system grows, it can be hard to visualise the dependencies between services. The Service Graph will draw a dependency tree of how the system connects.

https://2886795278-8088-cykoria08.environments.katacoda.com/dotviz


 

As Istio is managing the entire service-to-service communicate, the dashboard will highlight the aggregated totals and the breakdown on an individual service level.


 Visualise Cluster using Weave Scope

While Service Graph displays a high-level overview of how systems are connected, a tool called Weave Scope provides a powerful visualisation and debugging tool for the entire cluster.
Using Scope it's possible to see what processes are running within each pod and which pods are communicating with each other. This allows users to understand how Istio and their application is behaving.

Deploy Scope

Scope is deployed onto a Kubernetes cluster with the command


$kubectl create -f 'https://cloud.weave.works/launch/k8s/weavescope.yaml'
Wait for it to be deployed by checking the status of the pods using

$kubectl get pods -n weave




Make Scope Accessible

Once deployed, expose the service to the public.


$pod=$(kubectl get pod -n weave --selector=name=weave-scope-app -o jsonpath={.items..metadata.name})

$kubectl expose pod $pod -n weave --external-ip="172.17.0.60" --port=4040 --target-port=4040




View Scope on port 4040 at 
https://2886795278-4040-cykoria08.environments.katacoda.com/



Generate Load

Scope works by mapping active system calls to different parts of the application and the underlying infrastructure.

 Create load to see how various parts of the system now communicate.
.katacoda.com/productpage > /dev/null;   echo -n .;   sleep 0.2; done while true; do   curl -s https://2886795324-80-cykoria08.environments


while true; do   curl -s https://2886795324-80-cykoria08.environments.katacoda.com/productpage > /dev/null;   echo -n .;   sleep 0.2; done




