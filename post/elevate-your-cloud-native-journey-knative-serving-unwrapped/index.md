# Elevate your Cloud-Native Journey: Knative the VMware Tanzu Way Part II - Knative Serving unwrapped


## Knative Serving

Knative Serving is the runtime component of the Knative project. At its core, Knative Serving provides a Kubernetes-native platform to deploy, run, and manage modern `HTTP` workloads. In this second part of my series on Knative, I'll dive deep into the capabilities of Knative Serving and explore its unique features. Ultimately, I'll demonstrate why it's fast becoming the go-to choice for developers as well as platfrom engineers who want the flexibility of Kubernetes combined with the ease of serverless.

Whether you're a seasoned Kubernetes veteran or a newcomer looking to explore serverless options, there's something in Knative Serving for everyone.

In the context of Knative, serverless implies Knative's ability to abstracts away much of the underlying infrastructure complexity, allowing relevant teams to focus on writing code and defining the desired state of their  applications.

Key capabilities of Serving are:

- Serverless experience for `HTTP` workloads
- Scale-to-zero, request-driven compute runtime
- Point-in-time snapshot of your code and configuration (`Revision`)
- Traffic routing amongst `revisions`
- Supports deployment patterns like Blue/Green, Canary and Progressive
- Automatic DNS and TLS handling

## Not all Services are the Same

In Kubernetes, a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) is a fundamental resource used to define a network service and provide network connectivity to one or multiple pods. Kubernetes Services are essential for ensuring reliable communication and load balancing between different parts of your application within a cluster.

In Knative, a [Knative Service](https://github.com/knative/specs/blob/main/specs/serving/knative-api-specification-1.0.md#service) (`ksvc`) represents a part of your application or function. It's like a magic box for your applications that takes care of the technical details of your application, making it easier to run and manage it in a flexible and efficient way. When a `ksvc` gets created, it creates several other resources named `configuration`, a `revision` and `route`.

{{< image src="/img/posts/202310_knative_part_2/202310_knative_part_2_service.png" caption="Figure I: Knative Service Construct" src-s="/img/posts/202310_knative_part_2/202310_knative_part_2_service.png" >}}

*Source: [Knative Docs - Serving](https://knative.dev/docs/serving/#knative-serving)*

Knative Serving custom resources (CRs) in Kubernetes are:

- `services.serving.knative.dev` (`ksvc`)
- `configurations.serving.knative.dev` (`config`)
- `revisions.serving.knative.dev` (`rev`)
- `routes.serving.knative.dev` (`rt`)

## A Practical Comparison

Now that you know what's inside the magic box, what makes it so flexible and efficient? And what are the responsibilities of the mentioned CRs.

Let's take a closer look using a practical example. We want to deploy a really basic webserver application to Kubernetes **WITHOUT** using Knative. The webserver should be accessible on Layer-7 via `HTTPS`, which requires an Ingress resource and a configured `TLS` certificate. Furthermore, we want the Kubernetes [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) to scale our application based on compute utilization.

{{< admonition info "Out of Scope" true >}}
Custom metrics, DNS, traffic splitting and rollout-management.
{{< /admonition >}}

### How it is with Kubernetes

The first resource we're going to create the deployment itself which includes metadata like `name`, `annotations` and `labels` as well as certain configurations like `image` reference, # of `replicas` as well as `resource` specs.

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: knative-webapp
  name: knative-webapp
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: knative-webapp
  template:
    metadata:
      labels:
        app: knative-webapp
    spec:
      containers:
      - image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
        imagePullPolicy: IfNotPresent
        name: knative-webapp
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          limits:
            cpu: 1
            memory: 250Mi
EOF
```

In order to expose our app on the network, we will create an ingress resource. This requires an existing Ingress Controller like project [Contour](https://projectcontour.io). Before this resource gets created, we have to make sure that incoming network traffic will automatically be distributed among the pods associated with the service.

Therefore, a Kubernetes `service` type `ClusterIP` has to be created first.

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: knative-webapp
  name: knative-webapp-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: knative-webapp
  type: ClusterIP
EOF
```

For the sake of this example, I could leave TLS a side and just go with `HTTP` only, so no SSL/TLS termination at all. But I'd like to stay on the bright path. We need a `TLS` secret to secure communication over `HTTPS`. These secrets typically contain two pieces of data:

- `tls.crt`: This is the TLS certificate itself, including the server's public key.
- `tls.key`: This is the private key associated with the TLS certificate.

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: v1
type: kubernetes.io/tls
kind: Secret
metadata:
  name: knative-webapp-tls
data:
  tls.crt: <base64 encoded data>
  tls.key: <base64 encoded data>
EOF
```

The ingress object is a resource used to configure external access to services within a Kubernetes cluster. It serves as a way to manage and control `HTTP` and `HTTPS` traffic into the cluster. Besides SSL/TLS termination, it also provides features like routing and load balancing.

When configuring `HTTPS` for ingress resources, you can reference the `TLS` secret using the `tls.secretName` field. This allows you to specify which `TLS` certificate to use for a particular host or domain.

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  labels:
    app: knative-webapp
  name: knative-webapp-ingress
spec:
  tls:
  - hosts:
      - webapp.cpod-nsxv8.az-stc.cloud-garage.net
    secretName: knative-webapp-tls
  rules:
  - host: webapp.cpod-nsxv8.az-stc.cloud-garage.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: knative-webapp-svc
            port:
              number: 80
EOF
```

Last but not least, automatical scaling (horizontal) of our application. This is done by using the Kubernetes Horizontal Pod Autoscaler (HPA). In my provided configuration, the HPA will scale the app to a max of 3 replicas when an average CPU utilization of 2% is present for 15 seconds (default).

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: knative-webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: knative-webapp
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 2
EOF
```

If you successfully deployed all of the above, you should see the following resources within the namespace `myapp` are showing up.

```shell
kubectl -n myapp get po,deploy,svc,secret,ingress,hpa

NAME                                 READY   STATUS    RESTARTS   AGE
pod/knative-webapp-f75d4b9f6-2284j   1/1     Running   0          59m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/knative-webapp   1/1     1            1           59m

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/knative-webapp-svc   ClusterIP   10.96.131.102   <none>        80/TCP    58m

NAME                                                TYPE                DATA   AGE
secret/knative-webapp-tls                           kubernetes.io/tls   2      47m

NAME                                               CLASS    HOSTS                                       ADDRESS     PORTS     AGE
ingress.networking.k8s.io/knative-webapp-ingress   <none>   webapp.cpod-nsxv8.az-stc.cloud-garage.net   10.15.8.7   80, 443   48m

NAME                                                     REFERENCE                   TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         1          46m
```

Ultimately, your simple webserver should be available via the configured url. For me it's `https://webapp.cpod-nsxv8.az-stc.cloud-garage.net/`

Via **terminal**:
```shell
curl -kI https://webapp.cpod-nsxv8.az-stc.cloud-garage.net
HTTP/2 200
server: envoy
date: Mon, 09 Oct 2023 13:37:20 GMT
content-type: text/html
content-length: 1905
last-modified: Fri, 06 Oct 2023 07:06:18 GMT
etag: "651fb1ea-771"
accept-ranges: bytes
x-envoy-upstream-service-time: 26
vary: Accept-Encoding
```

Via **browser**:

{{< image src="/img/posts/202310_knative_part_2/202310_knative_part_2_k8s_kn_webapp.png" caption="Figure II: Simple Webserver Deployment on Kubernetes" src-s="/img/posts/202310_knative_part_2/202310_knative_part_2_k8s_kn_webapp.png" >}}

Reachability is validated but automatical scaling not yet. Let's put some load on it. The following example from the Kubernetes documentation can be greatly used:

`kubectl -n myapp run -i --tty load-generator --rm --image=projects.registry.vmware.com/tanzu_ese_poc/busybox:latest --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://webapp.cpod-nsxv8.az-stc.cloud-garage.net; done"`

{{< admonition info "HTTP used" true >}}
To avoid a TLS certificate validation error from the busybox pod, we'll use `HTTP` for the sake of simplicity.
{{< /admonition >}}

If you start watching (`-w`) what's going on in the `myapp` namespace, you'll see your workload scaling up and down automatically:

```shell
kubectl -n myapp get events -w

LAST SEEN   TYPE     REASON              OBJECT                                MESSAGE
0s          Normal    Started             pod/load-generator                    Started container load-generator
0s          Normal    SuccessfulRescale   horizontalpodautoscaler/knative-webapp-hpa   New size: 2; reason: cpu resource utilization (percentage of request) above target
0s          Normal    ScalingReplicaSet   deployment/knative-webapp                    Scaled up replica set knative-webapp-f75d4b9f6 to 2 from 1
0s          Normal    SuccessfulCreate    replicaset/knative-webapp-f75d4b9f6          Created pod: knative-webapp-f75d4b9f6-vkrl5
0s          Normal    Scheduled           pod/knative-webapp-f75d4b9f6-vkrl5           Successfully assigned myapp/knative-webapp-f75d4b9f6-vkrl5 to rguske-cnr-1-md-0-9z52m-5794dccc77-nnm8c
0s          Normal    Pulling             pod/knative-webapp-f75d4b9f6-vkrl5           Pulling image "projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1"
1s          Normal    Pulled              pod/knative-webapp-f75d4b9f6-vkrl5           Successfully pulled image "projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1" in 11.17134876s (11.171369522s including waiting)
0s          Normal    Created             pod/knative-webapp-f75d4b9f6-vkrl5           Created container knative-webapp
0s          Normal    Started             pod/knative-webapp-f75d4b9f6-vkrl5           Started container knative-webapp
0s          Normal    SuccessfulRescale   horizontalpodautoscaler/knative-webapp-hpa   New size: 1; reason: All metrics below target
1s          Normal    ScalingReplicaSet   deployment/knative-webapp                    Scaled down replica set knative-webapp-f75d4b9f6 to 1 from 2
0s          Normal    SuccessfulDelete    replicaset/knative-webapp-f75d4b9f6          Deleted pod: knative-webapp-f75d4b9f6-vkrl5
0s          Normal    Killing             pod/knative-webapp-f75d4b9f6-vkrl5           Stopping container knative-webapp
```

It's pretty nice to follow the events and to see how the emerging load is dealt with.

The operations can also be followed by requesting the `hpa` resource directly:

```shell
kubectl -n myapp get hpa knative-webapp-hpa --watch

NAME                 REFERENCE                   TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         1          26m
knative-webapp-hpa   Deployment/knative-webapp   1%/2%     1         3         1          33m
knative-webapp-hpa   Deployment/knative-webapp   2%/2%     1         3         1          34m
knative-webapp-hpa   Deployment/knative-webapp   2%/2%     1         3         1          34m
knative-webapp-hpa   Deployment/knative-webapp   1%/2%     1         3         1          34m
knative-webapp-hpa   Deployment/knative-webapp   1%/2%     1         3         1          35m
knative-webapp-hpa   Deployment/knative-webapp   4%/2%     1         3         1          35m
knative-webapp-hpa   Deployment/knative-webapp   2%/2%     1         3         2          35m
knative-webapp-hpa   Deployment/knative-webapp   1%/2%     1         3         2          36m
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         2          36m
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         2          37m
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         2          37m
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         2          37m
knative-webapp-hpa   Deployment/knative-webapp   1%/2%     1         3         2          38m
knative-webapp-hpa   Deployment/knative-webapp   0%/2%     1         3         2          38m
```

{{< admonition tip "Multiple load-generator instances" true >}}
If you loose patience and don't want to wait until the app gets scaled, simply run multiple busybox instances (`load-generator`, `load-generator-2`...).
{{< /admonition >}}

In summary one can say, it takes a lot of `yaml` to provide a small web service with required extras to be considered "production-ready".

### Knative - How easy it can be 🤯

Without any further ado, this is what you have to `apply` when using Knative:

**Declaratively:**

```yaml
kubectl -n myapp create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: knative-webapp-v1
spec:
  template:
    spec:
      containers:
      - image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
        ports:
        - containerPort: 80
          protocol: TCP
EOF
```

And it is even less **imperatively**:

```shell
### Creating Knative Service using kn cli
kn -n myapp service create knative-webapp --image projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1 --port 80

Warning: Kubernetes default value is insecure, Knative may default this to secure in a future release: spec.template.spec.containers[0].securityContext.allowPrivilegeEscalation, spec.template.spec.containers[0].securityContext.capabilities, spec.template.spec.containers[0].securityContext.runAsNonRoot, spec.template.spec.containers[0].securityContext.seccompProfile
Creating service 'knative-webapp' in namespace 'myapp':

  0.699s The Route is still working to reflect the latest desired specification.
  0.876s Configuration "knative-webapp" is waiting for a Revision to become ready.
 26.558s ...
 27.252s Ingress has not yet been reconciled.
 27.445s Waiting for Envoys to receive Endpoints data.
 28.350s Waiting for load balancer to be ready
 29.078s Certificate route-6a3f0955-7515-421d-9d01-0a595b510839 is not ready.
 29.690s Ingress has not yet been reconciled.
 29.961s Waiting for Envoys to receive Endpoints data.
 31.172s Waiting for load balancer to be ready
 31.532s Ready to serve.

Service 'knative-webapp' created to latest revision 'knative-webapp-00001' is available at URL:
https://knative-webapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net
```

How awesome is that? :rocket: AND! Knative provides **out-of-the-box scale-to-0 and scale-to-n**.






- **Scaling**: It makes sure your workload can handle as many incoming requests as needed. The more incoming traffic reaches your workload, the more it'll be scaled. Depending on `--max-scale`.  When nobody's around, it makes some servers disappear to save resources.
- **URL**: It gives your application a special web address (URL) that doesn't change, even when your application changes. This means people can always find your app at the same place on the internet.
- **Traffic Control**: It helps you control how traffic flows to different versions of your app. You can gradually roll out new features or switch back to older ones without people even noticing.
- **Easy Updates**: When you want to update your app, you just put the new version in the magic box. It takes care of the rest, making sure the transition is smooth and without any downtime.

- **Configuration**: A Knative Service resource includes configuration settings for your application or function. This includes information like container image details, environment variables, and build specifications.
- **Auto-scaling**: Knative Services automatically handle the scaling of your application or function based on incoming traffic. You can define scaling policies within the Knative Service resource to control how and when the service scales.
- **Route Configuration**: Knative Services often include route configuration, which defines how traffic is routed to different revisions of your application. This enables canary deployments, blue-green deployments, and traffic splitting between different versions.
- **Service Discovery**: Knative Services come with built-in service discovery mechanisms, making it easy for other services and external clients to discover and interact with your application or function.
- **Revision Management**: Knative Services allow you to manage different revisions of your application, which can be useful for rollbacks or A/B testing. Each revision corresponds to a specific deployment of your application.
- **Event Triggers**: You can also configure event triggers within a Knative Service resource, allowing your service to respond to external events or changes in the environment.
- **Annotations and Labels**: Knative Services can be annotated and labeled to provide additional metadata and information about the service.


```yaml
kubectl -n myapp get configurations.serving.knative.dev knative-webapp -oyaml

apiVersion: serving.knative.dev/v1
kind: Configuration
metadata:
  annotations:
    serving.knative.dev/creator: sso:Administrator@cpod-nsxv8.az-stc.cloud-garage.net
    serving.knative.dev/lastModifier: sso:Administrator@cpod-nsxv8.az-stc.cloud-garage.net
    serving.knative.dev/routes: knative-webapp
  creationTimestamp: "2023-10-09T21:36:21Z"
  generation: 1
  labels:
    serving.knative.dev/service: knative-webapp
    serving.knative.dev/serviceUID: 0c2ac867-693d-4acc-80a2-df9db8d182e0
  name: knative-webapp
  namespace: myapp
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: knative-webapp
    uid: 0c2ac867-693d-4acc-80a2-df9db8d182e0
  resourceVersion: "7746466"
  uid: 748fc5d8-79b3-47fa-8311-37d1e60ef120
spec:
  template:
    metadata:
      annotations:
        client.knative.dev/updateTimestamp: "2023-10-09T21:36:21Z"
        client.knative.dev/user-image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
      creationTimestamp: null
    spec:
      containerConcurrency: 0
      containers:
      - image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
        name: user-container
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          successThreshold: 1
          tcpSocket:
            port: 0
        resources: {}
      enableServiceLinks: false
      timeoutSeconds: 300
status:
  conditions:
  - lastTransitionTime: "2023-10-09T21:36:48Z"
    status: "True"
    type: Ready
  latestCreatedRevisionName: knative-webapp-00001
  latestReadyRevisionName: knative-webapp-00001
  observedGeneration: 1
```



```yaml
kubectl -n myapp get revisions.serving.knative.dev knative-webapp-00001 -oyaml
apiVersion: serving.knative.dev/v1
kind: Revision
metadata:
  annotations:
    client.knative.dev/updateTimestamp: "2023-10-09T21:36:21Z"
    client.knative.dev/user-image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
    serving.knative.dev/creator: sso:Administrator@cpod-nsxv8.az-stc.cloud-garage.net
    serving.knative.dev/routes: knative-webapp
    serving.knative.dev/routingStateModified: "2023-10-09T21:36:22Z"
  creationTimestamp: "2023-10-09T21:36:22Z"
  generation: 1
  labels:
    serving.knative.dev/configuration: knative-webapp
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 748fc5d8-79b3-47fa-8311-37d1e60ef120
    serving.knative.dev/routingState: active
    serving.knative.dev/service: knative-webapp
    serving.knative.dev/serviceUID: 0c2ac867-693d-4acc-80a2-df9db8d182e0
  name: knative-webapp-00001
  namespace: myapp
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Configuration
    name: knative-webapp
    uid: 748fc5d8-79b3-47fa-8311-37d1e60ef120
  resourceVersion: "7749777"
  uid: 41ae1fe0-86e5-4281-8bc5-f1f31b91fc75
spec:
  containerConcurrency: 0
  containers:
  - image: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp:v1
    name: user-container
    ports:
    - containerPort: 80
      protocol: TCP
    readinessProbe:
      successThreshold: 1
      tcpSocket:
        port: 0
    resources: {}
  enableServiceLinks: false
  timeoutSeconds: 300
status:
  actualReplicas: 0
  conditions:
  - lastTransitionTime: "2023-10-09T21:40:37Z"
    message: The target is not receiving traffic.
    reason: NoTraffic
    severity: Info
    status: "False"
    type: Active
  - lastTransitionTime: "2023-10-09T21:36:48Z"
    status: "True"
    type: ContainerHealthy
  - lastTransitionTime: "2023-10-09T21:36:48Z"
    status: "True"
    type: Ready
  - lastTransitionTime: "2023-10-09T21:36:24Z"
    status: "True"
    type: ResourcesAvailable
  containerStatuses:
  - imageDigest: projects.registry.vmware.com/tanzu_ese_poc/rguske/knative-webapp@sha256:fd918309bb55c5a0d8877aa3c2786da5cd12e30408accadddcc0b778d43b4019
    name: user-container
  desiredReplicas: 0
  observedGeneration: 1
```


```yaml
kubectl -n myapp get routes.serving.knative.dev knative-webapp -oyaml
apiVersion: serving.knative.dev/v1
kind: Route
metadata:
  annotations:
    serving.knative.dev/creator: sso:Administrator@cpod-nsxv8.az-stc.cloud-garage.net
    serving.knative.dev/lastModifier: sso:Administrator@cpod-nsxv8.az-stc.cloud-garage.net
  creationTimestamp: "2023-10-09T21:36:22Z"
  finalizers:
  - routes.serving.knative.dev
  generation: 1
  labels:
    serving.knative.dev/service: knative-webapp
  name: knative-webapp
  namespace: myapp
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: knative-webapp
    uid: 0c2ac867-693d-4acc-80a2-df9db8d182e0
  resourceVersion: "7746612"
  uid: 6a3f0955-7515-421d-9d01-0a595b510839
spec:
  traffic:
  - configurationName: knative-webapp
    latestRevision: true
    percent: 100
status:
  address:
    url: http://knative-webapp.myapp.svc.cluster.local
  conditions:
  - lastTransitionTime: "2023-10-09T21:36:48Z"
    status: "True"
    type: AllTrafficAssigned
  - lastTransitionTime: "2023-10-09T21:36:51Z"
    status: "True"
    type: CertificateProvisioned
  - lastTransitionTime: "2023-10-09T21:36:53Z"
    status: "True"
    type: IngressReady
  - lastTransitionTime: "2023-10-09T21:36:53Z"
    status: "True"
    type: Ready
  observedGeneration: 1
  traffic:
  - latestRevision: true
    percent: 100
    revisionName: knative-webapp-00001
  url: https://knative-webapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net
```



{{< admonition info "" true >}}

{{< /admonition >}}


Knative Services automatically scale up or down based on incoming traffic. When there are no requests, it can scale to zero, effectively reducing resource consumption and costs. As requests increase, it can scale up to handle the load, ensuring optimal resource utilization.



| | | |
|:---: |:---: |:---: | :---: |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |


{{< mermaid >}}
graph LR;
    subgraph HAProxy Appliance
    A[ Management ]
    B[ Workload ]
    C[ Frontend / VIPs ]
    end
    subgraph Traffic from
    G( Client / Service )
    end
    subgraph Load Balancer VIPs
    E[ Supervisor Cluster VMs ]
    F[ Guest Cluster VMs ]
    end
    A & B & C --- E
    B & C --- F
    G -.- C -.-> F & E
{{< /mermaid >}}
