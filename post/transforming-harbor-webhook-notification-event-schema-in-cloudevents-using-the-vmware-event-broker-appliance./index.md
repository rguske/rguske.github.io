# Transforming Harbor Webhook Notification Event Schema in CloudEvents using the VMware Event Broker Appliance


<!--more-->

## Web🪝s, Web🪝s, Web🪝s

Many solutions today are supporting the configuration of a webhook endpoint, a remote webserver, to send information to it over `HTTP, HTTPS POST/PUT`. Information as such can vary but a popular use case for example is to send out notifications in case of the occurrence of a specific event. The information sent are often provided in a `JSON` structured format. The open source container registry [Harbor](https://goharbor.io/) for instance, which I already tackled a couple of times on my blog ([#harbor](https://rguske.github.io/tags/harbor/)), supports the configuration of a webhook endpoint in order to send specific notification events via `http` or `https` (`POST`) as well. Those events are related to a created [Harbor project](https://goharbor.io/docs/latest/working-with-projects/create-projects/) specifically.

{{< admonition info "Harbor project" true >}}
A project in Harbor contains all repositories of an application. Images cannot be pushed to Harbor before a project is created. Role-Based Access Control (RBAC) is applied to projects, so that only users with the appropriate roles can perform certain operations.

*Source: [Harbor Documentation](https://goharbor.io/docs/latest/working-with-projects/create-projects/)*
{{< /admonition >}}

When now a certain project related interaction happened, like e.g. the `push` of a new container image (`PUSH_ARTIFACT`) or an image got pulled (`PULL_ARTIFACT`) or deleted (`DELETE_ARTIFACT`) to/from the project, the following payload will be send to the configured webhook endpoint:

```json
// Harbor specific JSON payload

{
  "type": "DELETE_ARTIFACT",
  "occur_at": 1655887039,
  "operator": "admin",
  "event_data": {
    "resources": [
      {
        "digest": "sha256:3b465cbcadf7d437fc70c3b6aa2c93603a7eef0a3f5f1e861d91f303e4aabdee",
        "tag": "v2.2.2",
        "resource_url": "harbor-app.jarvis.tanzu/veba-test/veba-image:v1.0"
      }
    ],
    "repository": {
      "name": "veba-image",
      "namespace": "veba-test",
      "repo_full_name": "veba-test/veba-image",
      "repo_type": "public"
    }
  }
}
```

*Source: [Harbor Documentation](https://goharbor.io/docs/latest/working-with-projects/project-configuration/configure-webhooks/)*

The information sent are very useful and are serving the receiver with details like e.g. `which`, `where`, `who`, `when` and more. Now, since those information are very, again useful as well as relevant, we want to process such in the most effective way as possible. This is where I came up with the idea to use the [VMware Event Broker Appliance - VEBA](https://vmweventbroker.io) for it, in order to ensure effectiveness.

## VEBA Generic Web🪝 Endpoint and Web🪝 Functions

VEBA supports a generic webhook endpoint since version [v0.7.0](https://vmweventbroker.io/evolution), which is configureable during the OVA deployment and reachable afterwards on `https://<veba-fqdn>/webhook`. This endpoint can be used to further expand the range of event producers who supports the creation of their own event payload.

[William Lam](https://twitter.com/lamw) introduced a webhook function in his article ["Custom webhook function to publish events into VMware Event Broker Appliance (VEBA)"](https://williamlam.com/2021/09/custom-webhook-function-to-publish-events-into-vmware-event-broker-appliance-veba.html) and covers the awesomeness and simplicity when leveraging VEBA functions in order to transform a non-CloudEvent conformant payload and to ultimately enable the integration of other (non-vSphere/Horizon) systems/event producers.

Now the challenge is, and William directly tackled it at the very beginning of his post, that not every solution supports the [CloudEvents](https://cloudevents.io/) specification/schema. The idea behind CloudEvents is simple and powerful at the same time. If there's no default description/definition for event data, event handling logics have to be developed again and again!

I hope this will change in the future soon ...

<center> {{< tweet user="embano1" id="1527298365528190976" >}} </center>

> Session from the tweet: **"Thinking Cloud Native, CloudEvents Future - Scott Nichols, Chainguard"** at KubeCon 2022 Europe :point_down: [Resources](#resources).

In my specific use case with Harbor, I checked the open issue's on the project page on Github, to see if someone already raised the idea of making the webhook notification events CloudEvent conform and yes, <i class='fab fa-github fa-fw'></i> [#10146](https://github.com/goharbor/harbor/issues/10146) is bringing this idea up. Give it a :thumbs_up: :wink:

However, in order to react on the sent events from Harbor, I briefly brainstormed with [Michael](https://twitter.com/embano1), got him hooked on the overall idea, and he quickly wrote a complete new function which transforms the incoming Harbor events into CloudEvents :boom:.

<center> {{< tweet user="embano1" id="1541387083515846658" >}} </center>

## The Use Case with Harbor

So, the overall idea is to send the non-CloudEvent payload to the new `kn-go-harbor-webhook` function, get it transformed into a CloudEvent payload and have the new [kn-ps-harbor-slack](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/powershell/kn-ps-harbor-slack) function triggered on the delivered and `subscribed` event like e.g. `com.vmware.harbor.push_artifact.v0`.

This is how the flow looks like when you've deployed it:

{{< mermaid >}}
graph TD;
    subgraph Harbor Registry
    A
    end
    subgraph VEBA
    B
    C
    D
    end
    subgraph Slack
    E
    end
    A[Webhook Notification] -->|Harbor Event Payload| B[kn-go-harbor-webhook Function]
    B -->|CloudEvent Transformation - New Payload| C[Knative Magic]
    D[kn-ps-harbor-slack Function] --> C
    D --> E[Slack Channel Webhook]
{{< /mermaid >}}

{{< admonition info "Low hanging Fruits" true >}}
:cherries: = Existing [VEBA Function Examples](https://vmweventbroker.io/examples)

In order to not only have a "transformation" function available but also a function which will be triggered on a new Harbor specific event, I simply took the existing [Slack-Function example](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/master/examples/knative/powershell/kn-ps-slack), adjusted the necessary `fields` whithin the `handler.ps1` accordingly in order to ultimately pick the wanted data from the incoming event.

Voilà, [kn-ps-harbor-slack](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/powershell/kn-ps-harbor-slack) is ready to serve.
{{< /admonition >}}

## Deploying the new Harbor Functions

Let me guide you through the necessary steps in order to:

1. have a properly prepared and configured environment (DNS, VEBA, Harbor) available
2. have the [kn-go-harbor-webhook](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/go/kn-go-harbor-webhook) function deployed
3. have the [kn-ps-harbor-slack](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/powershell/kn-ps-harbor-slack) function deployed

### DNS Wildcard Configuration for VEBA

The first required step is to configure your DNS server with a wildcard entry for VEBA. This is necessary because when you're going to deploy functions to VEBA, they'll ultimately run as Pods on Kubernetes, or in Knative terminology as [Knative Services](https://knative.dev/docs/serving/services/) (`serving.knative.dev/service`). Those Services are providing endpoints for which we have to make sure that they are reachable over the wire.

It is presented as follows: `https://[function-name].[function-namespace].[veba-fqdn]`.

Since we are talking about Kubernetes Custom Resources, we can interact with such using `kubectl`. In order to e.g. check your functions, run `kubectl -n vmware-functions get ksvc`.

I'm using the Knative CLI for it because it'll add an additional column to the output which is `CONDITIONS`. It's always good to check those.

```shell
kn service list -n vmware-functions

NAME                   URL                                                                  LATEST                       AGE    CONDITIONS   READY   REASON
kn-go-harbor-webhook   http://kn-go-harbor-webhook.vmware-functions.veba-dev.jarvis.tanzu   kn-go-harbor-webhook-00001   9d     3 OK / 3     True
kn-pcli-nsx-tag-sync   http://kn-pcli-nsx-tag-sync.vmware-functions.veba-dev.jarvis.tanzu   kn-pcli-nsx-tag-sync-00001   13d    3 OK / 3     True
kn-pcli-tag            http://kn-pcli-tag.vmware-functions.veba-dev.jarvis.tanzu            kn-pcli-tag-00001            133d   3 OK / 3     True
kn-ps-harbor-slack     http://kn-ps-harbor-slack.vmware-functions.veba-dev.jarvis.tanzu     kn-ps-harbor-slack-00001     9d     3 OK / 3     True
```

Configuring a DNS wildcard entry on a Windows Server is made easy. Open the **DNS Manager**, right-click on your **Forward Lookup Zone** and select **New Host (A or AAA)**. For **Name** enter `*.<VEBA-Hostname>`. This should automatically create a FQDN like `*.veba-dev.jarvis.tanzu` in my example. Last is the **IP address** of your VEBA.

{{< image src="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-1.png" caption="Figure I: Windows Server Wildcard DNS Configuration" src-s="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-1.png" >}}

William is explaining the necessary configurations on his blog when using Unbound on Ubuntu - [LINK](https://williamlam.com/2021/09/custom-webhook-function-to-publish-events-into-vmware-event-broker-appliance-veba.html).

### Deploy the kn-go-harbor-webhook Function

Deploying functions on VEBA is simple and well documented guidance for each specific example function is available on <i class='fab fa-github fa-fw'></i>. If necessary, the first step is in most cases the creation of a Kubernetes secret in order to store sensible data like e.g. user credentials or a webhook URL on Kubernetes. Having said, we are going to create a new secret called `webhook-auth` for our function in order to enforce basic authentication on the HTTP endpoint.

1. Create the `webhook-auth` secret

```shell
kubectl create secret generic webhook-auth \
--type=kubernetes.io/basic-auth \
--from-literal=username='webhookuser' \
--from-literal=password='replaceme' \
--namespace vmware-functions
```

Checking the secret to see if the given values have been passed.

```shell
kubectl -n vmware-functions get secret webhook-auth -o yaml
apiVersion: v1
data:
  password: cmVwbGFjZW1l
  username: d2ViaG9va3VzZXI=
kind: Secret
metadata:
  creationTimestamp: "2022-07-04T14:05:49Z"
  name: webhook-auth
  namespace: vmware-functions
  resourceVersion: "291669370"
  uid: c0fc6ecc-a6cb-4532-813b-daea79dc0604
type: kubernetes.io/basic-auth
```

Let's quickly decrypt the value for `password`:

```shell
echo cmVwbGFjZW1l | base64 -d

replaceme
```

Compliant :thumbs_up:

2. Create a `SinkBinding`

A Knative `SinkBinding` makes the decoupling of an `event source` (like Harbor) and an `event sink` (Knative Broker) possible. Therefore, it's possible to automatically inject the VEBA default broker into the function.

Create the `SinkBindung`:

```yaml
kubectl -n vmware-functions create -f - <<EOF
apiVersion: sources.knative.dev/v1
kind: SinkBinding
metadata:
  name: kn-go-harbor-webhook-binding
spec:
  subject:
    apiVersion: serving.knative.dev/v1
    kind: Service
    name: kn-go-harbor-webhook
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
EOF
```

When checking the deployed object by executing `kubectl -n vmware-functions get sinkbinding`, you'll probably notice the info `REASON: SubjectMissing`. This is normal since it is waiting for the actual function deployment.

3. Deploy the kn-go-harbor-webhook function

The final step now is the deployment of the `kn-go-harbor-webhook` function itself.

```yaml
kubectl -n vmware-functions create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: kn-go-harbor-webhook
  labels:
    app: veba-ui
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "1"
        autoscaling.knative.dev/minScale: "1"
    spec:
      containers:
        - image: us.gcr.io/daisy-284300/veba/kn-go-harbor-webhook:1.0
          imagePullPolicy: IfNotPresent
          env:
            - name: ADDRESS
              value: "0.0.0.0"
            - name: WEBHOOK_PATH
              value: "/webhook" # default
            - name: DEBUG
              value: "true"
            - name: WEBHOOK_SECRET_PATH # remove this env to disable basic auth
              value: "/var/bindings/webhook"
          volumeMounts: # remove this when not using basic auth
            - name: webhook-auth
              mountPath: "/var/bindings/webhook"
              readOnly: true
      volumes: # remove this when not using basic auth
        - name: webhook-auth
          secret:
            secretName: webhook-auth
EOF
```

Conditions and state of the deployment can be checked as already described using `kubectl` or `kn`.

```shell
# Using kubectl
kubectl -n vmware-functions get ksvc
NAME                   URL                                                                  LATESTCREATED                LATESTREADY                  READY   REASON
kn-go-harbor-webhook   http://kn-go-harbor-webhook.vmware-functions.veba-dev.jarvis.tanzu   kn-go-harbor-webhook-00001   kn-go-harbor-webhook-00001   True

# Using kn
kn -n vmware-functions service list
NAME                   URL                                                                  LATEST                       AGE    CONDITIONS   READY   REASON
kn-go-harbor-webhook   http://kn-go-harbor-webhook.vmware-functions.veba-dev.jarvis.tanzu   kn-go-harbor-webhook-00001   78m    3 OK / 3     True
```

Also, checking the logs tells you a lot.

```shell
kubectl -n vmware-functions logs kn-go-harbor-webhook-00001-deployment-7d76fbb5fc-qclfv user-container

2022-07-04T14:32:14.599Z        INFO    harbor-webhook  kn-go-harbor-webhook/server.go:95       starting http server    {"commit": "5472048a", "tag": "1.0", "address": "0.0.0.0:8080", "path": "/webhook", "sink": "http://default-broker-ingress.vmware-functions.svc.cluster.local", "basic_auth": true}
```

Well, everything is in good condition and the function is ready to receive and transform Harbor notification events.

### Configure the Webhook Feature in Harbor

In Harbor, open your project and go to **Webhooks**. Click on **+ NEW WEBHOOK** and configure it to meet your requirements (e.g. in terms of event types). Important is the **Endpoint URL** as well as the **Auth Header**.

For **Endpoint URL**, enter the complete URL ending with `/webhook` of the recently created kn-go-harbor-webhook function/Knative Service. For me it's: `https://kn-go-harbor-webhook.vmware-functions.veba-dev.jarvis.tanzu/webhook`

For **Auth Header**, enter `Basic` followed by a whitespace and the base64-encoded value of the `<username>:<password>` string combination defined in the `webhook-auth` secret step. Use a Basic Auth Header Generator like e.g. [DebugBear](https://www.debugbear.com/basic-auth-header-generator).

{{< image src="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-2.png" caption="Figure II: Webhook Configuration in Harbor" src-s="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-2.png" >}}

The Webhook config should show up as shown in *Firgue III* below:

{{< image src="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-3.png" caption="Figure III: Configured Webhook in Harbor" src-s="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-3.png" >}}

### Verifying the Configuration

Verifying if everything works as expected can be done by e.g. `push`ing an image to your project (repo) or e.g. `delete`ing an existing one. For the sake of my example, I deleted an existing image (`artifact`) and checked the logs of the `user-container`, which runs inside the `kn-ps-harbor-slack-00001-deployment-xxx-xxx` pod.

```shell
$ kubectl -n vmware-functions logs kn-go-harbor-webhook-00001-deployment-7d76fbb5fc-qclfv user-container -f

2022-07-04T14:32:14.599Z        INFO    harbor-webhook  kn-go-harbor-webhook/server.go:95       starting http server    {"commit": "5472048a", "tag": "1.0", "address": "0.0.0.0:8080", "path": "/webhook", "sink": "http://default-broker-ingress.vmware-functions.svc.cluster.local", "basic_auth": true}
2022-07-04T15:01:02.406Z        DEBUG   harbor-webhook  kn-go-harbor-webhook/server.go:140      received request        {"commit": "5472048a", "tag": "1.0", "eventID": "f5c27ced-2b6b-45c3-a795-59efc33947b0", "request": "{\"type\":\"DELETE_ARTIFACT\",\"occur_at\":1656946862,\"operator\":\"admin\",\"event_data\":{\"resources\":[{\"digest\":\"sha256:d3890814cc5a7cfc02403435281cdf51adfb6b67e223934d9d6137a4ad364286\",\"tag\":\"1.21.6-debian-10-r117\",\"resource_url\":\"harbor.jarvis.tanzu/veba-webhook/bitnami-nginx:1.21.6-debian-10-r117\"}],\"repository\":{\"name\":\"bitnami-nginx\",\"namespace\":\"veba-webhook\",\"repo_full_name\":\"veba-webhook/bitnami-nginx\",\"repo_type\":\"public\"}}}"}
2022-07-04T15:01:02.412Z        DEBUG   harbor-webhook  kn-go-harbor-webhook/server.go:173      successfully sent cloudevent    {"commit": "5472048a", "tag": "1.0", "eventID": "f5c27ced-2b6b-45c3-a795-59efc33947b0", "event": "Context Attributes,\n  specversion: 1.0\n  type: com.vmware.harbor.delete_artifact.v0\n  source: /kn-go-harbor-webhook\n  subject: admin\n  id: f5c27ced-2b6b-45c3-a795-59efc33947b0\n  time: 2022-07-04T15:01:02Z\n  datacontenttype: application/json\nData,\n  {\n    \"type\": \"DELETE_ARTIFACT\",\n    \"occur_at\": 1656946862,\n    \"operator\": \"admin\",\n    \"event_data\": {\n      \"resources\": [\n        {\n          \"digest\": \"sha256:d3890814cc5a7cfc02403435281cdf51adfb6b67e223934d9d6137a4ad364286\",\n          \"tag\": \"1.21.6-debian-10-r117\",\n          \"resource_url\": \"harbor.jarvis.tanzu/veba-webhook/bitnami-nginx:1.21.6-debian-10-r117\"\n        }\n      ],\n      \"repository\": {\n        \"name\": \"bitnami-nginx\",\n        \"namespace\": \"veba-webhook\",\n        \"repo_full_name\": \"veba-webhook/bitnami-nginx\",\n        \"repo_type\": \"public\"\n      }\n    }\n  }\n"}
```

A beautified view is available in VEBA's provided event viewer via `https://<veba-fqdn>/events`.

{{< image src="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-4.png" caption="Figure IV: Harbor CloudEvent in Sockeye" src-s="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-4.png" >}}

Awesome! :rocket:

:movie_camera: **kn-go-harbor-webhook function deployment** :movie_camera:

<script id="asciicast-qECs6T0ulUiXNIchr3UFcLEr3" src="https://asciinema.org/a/qECs6T0ulUiXNIchr3UFcLEr3.js" async></script>

### Deploy the kn-ps-harbor-slack Function

With the available CloudEvent payload, we can now configure other functions for being subscribed to incoming events and to ultimately get triggered by those. As aforementioned (low hanging fruits), I wanted to get notified by actions happened related to my Harbor project. The [kn-ps-harbor-slack](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/powershell/kn-ps-harbor-slack) function will send a good looking message to your Slack channel webhook (there we go again... web:hook:s everywhere :smile:).

1. Add your Slack channel webhook URL to the [slack_secret.json](https://github.com/vmware-samples/vcenter-event-broker-appliance/blob/development/examples/knative/powershell/kn-ps-harbor-slack/slack_secret.json) file
2. Create the Kubernetes secret

```shell
# Create the harbor-slack-secret
$ kubectl -n vmware-functions create secret generic harbor-slack-secret --from-file=SLACK_SECRET=slack_secret.json

# Verify the creation of the secret
$ kubectl -n vmware-functions get secrets

NAME                    TYPE                                  DATA   AGE
harbor-slack-secret     Opaque                                1      1m
webhook-auth            kubernetes.io/basic-auth              2      9m
```

3. Deploy the `function.yaml`

By default, the function deployment will filter on the `com.vmware.harbor.push_artifact.v0` Harbor Event. If you wish to change this, update the `type` field within `function.yaml` to the desired event type. A list of supported notification events is available on the official Harbor documentation under [Configure Webhook Notifications](https://goharbor.io/docs/latest/working-with-projects/project-configuration/configure-webhooks/). As mentioned, use the VEBA event viewer endpoint (`https://<veba-fqdn>/events`) to display all incoming events.

```yaml
kubectl -n vmware-functions create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: kn-ps-harbor-slack
  labels:
    app: veba-ui
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "1"
        autoscaling.knative.dev/minScale: "1"
    spec:
      containers:
        - image: us.gcr.io/daisy-284300/veba/kn-ps-harbor-slack:1.0
          envFrom:
            - secretRef:
                name: harbor-slack-secret
          env:
            - name: FUNCTION_DEBUG
              value: "false"
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: kn-ps-harbor-slack-trigger
  labels:
    app: veba-ui
spec:
  broker: default
  filter:
    attributes:
      type: com.vmware.harbor.push_artifact.v0
      subject: admin
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: kn-ps-harbor-slack
EOF
```

When you now repeat one of the described steps from above, like e.g. `push` or `delete` an artifact, a new message should show up in your Slack channel, like my provided example in *Firgure V*.

{{< image src="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-5.png" caption="Figure V: Slack Notification delivered by kn-ps-harbor-slack Function" src-s="/img/posts/202207_harbor_webhhok_post/rguske-post-harbor-webhook-function-5.png" >}}

:movie_camera: **kn-ps-harbor-slack function deployment** :movie_camera:

<script id="asciicast-WMgGk65yfymhqAQFkEQfRhQ3S" src="https://asciinema.org/a/WMgGk65yfymhqAQFkEQfRhQ3S.js" async></script>

**Sitenote:** Harbor supports Slack as a configurable [Notify Type](https://goharbor.io/docs/2.5.0/working-with-projects/project-configuration/configure-webhooks/) OOB but it sends the message in a raw JSON format. However, we also have a [Microsoft Teams Function example](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/knative/powershell/kn-ps-ngw-teams) available in PowerShell as well :wink:

## Resources

[^1]: [Thinking Cloud Native, CloudEvents Future - Scott Nichols, Chainguard](https://www.youtube.com/watch?v=Y6D0AY5aK-4)

{{< youtube Y6D0AY5aK-4 >}}
