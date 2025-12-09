# Kubevirtmeetseventing

## Introduction


```yaml
oc create -f - <<EOF
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: broker-apiserversource
spec: {}
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: broker-eventtransform
spec: {}
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: events-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: event-watcher
rules:
  - apiGroups:
      - "kubevirt.io"
    resources:
      - virtualmachines
      - virtualmachineinstances
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: k8s-ra-event-watcher
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: event-watcher
subjects:
  - kind: ServiceAccount
    name: events-sa
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: sources.knative.dev/v1
kind: ApiServerSource
metadata:
  name: apiserversource
  labels:
    app: apiserversource
spec:
  mode: Resource
  resources:
    - apiVersion: kubevirt.io/v1
      kind: VirtualMachine
  serviceAccountName: events-sa
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: broker-apiserversource
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-display
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/max-scale: "1"
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containerConcurrency: 0
      containers:
      - image: quay.io/openshift-knative/showcase
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-apiserversource
  name: trigger-event-display
spec:
  broker: broker-apiserversource
  filter: {}
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
EOF
```

```yaml
oc create -f - <<EOF
---
apiVersion: pool.kubevirt.io/v1alpha1
kind: VirtualMachinePool
metadata:
  name: dev-eventing-vms
spec:
  replicas: 0
  selector:
    matchLabels:
      kubevirt.io/vmpool: dev-eventing-vms
  virtualMachineTemplate:
    metadata:
      labels:
        kubevirt.io/vmpool: dev-eventing-vms
    spec:
      runStrategy: Always
      template:
        metadata:
          labels:
            kubevirt.io/vmpool: dev-eventing-vms
        spec:
          domain:
            cpu:
              cores: 1
            devices:
              disks:
              - disk:
                  bus: virtio
                name: containerdisk
              - disk:
                  bus: virtio
                name: cloudinitdisk
              interfaces:
                - masquerade: {}
                  model: virtio
                  name: default
              rng: {}
            memory:
              guest: 1024M
          networks:
            - name: default
              pod: {}
          volumes:
          - containerDisk:
              image: quay.io/containerdisks/fedora:latest
            name: containerdisk
          - cloudInitNoCloud:
              userData: |-
                #cloud-config
                password: fedora
                chpasswd: { expire: False }
            name: cloudinitdisk
EOF
```

`oc scale vmpool dev-eventing-vms --replicas 1`

```yaml
kubectl create -f - <<EOF
apiVersion: eventing.knative.dev/v1alpha1
kind: EventTransform
metadata:
  name: vmdata-transform
spec:
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: broker-eventtransform
  jsonata:
    expression: |
      {
        "specversion": specversion,
        "type": type,
        "source": source,
        "subject": subject,
        "id": id,
        "time": time,
        "kind": kind,
        "name": name,
        "namespace": namespace,
        "cpucores": data.spec.template.spec.domain.cpu.cores,
        "cpusockets": data.spec.template.spec.domain.cpu.sockets,
        "memory": data.spec.template.spec.domain.memory.guest,
        "datasource": data.spec.dataVolumeTemplates.spec.storage.resources.resources.storage,
        "storageclass": data.spec.dataVolumeTemplates.spec.storage.storageClassName,
        "network": data.spec.template.spec.networks.name
      }
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-apiserversource
  name: trigger-transformer-vm-add
spec:
  broker: broker-apiserversource
  filter:
    attributes:
      type: dev.knative.apiserver.resource.add
  subscriber:
    ref:
      apiVersion: eventing.knative.dev/v1alpha1
      kind: EventTransform
      name: vmdata-transform
  delivery:
    retry: 1
    backoffPolicy: linear
    backoffDelay: PT5S
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-apiserversource
  name: trigger-transformer-vm-delete
spec:
  broker: broker-apiserversource
  filter:
    attributes:
      type: dev.knative.apiserver.resource.delete
  subscriber:
    ref:
      apiVersion: eventing.knative.dev/v1alpha1
      kind: EventTransform
      name: vmdata-transform
  delivery:
    retry: 1
    backoffPolicy: linear
    backoffDelay: PT5S
EOF
```

```yaml
oc create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  labels:
    eventing.knative.dev/broker: broker-eventtransform
  name: sockeye
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/max-scale: "1"
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containerConcurrency: 0
      containers:
      - image: n3wscott/sockeye:v0.7.0
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-eventtransform
  name: trigger-sockeye
spec:
  broker: broker-eventtransform
  filter: {}
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: sockeye
EOF
```

## PostgreSQL DB StatefulSet


## The Function Deployment


```code
kubectl create secret generic psql-secret \
  --from-literal=db_host="changeme" \
  --from-literal=db_port="changeme" \
  --from-literal=db_name="changeme" \
  --from-literal=db_user="changeme" \
  --from-literal=db_password="changeme"
```


```yaml
oc create -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: kn-py-psql-vmdata-fn
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "1"
        autoscaling.knative.dev/minScale: "1"
    spec:
      containers:
        - image: quay.io/rguske/kn-py-psql-vmdata-fn:v1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: psql-secret
                  key: db_host
            - name: DB_PORT
              valueFrom:
                secretKeyRef:
                  name: psql-secret
                  key: db_port
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: psql-secret
                  key: db_name
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: psql-secret
                  key: db_user
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: psql-secret
                  key: db_password
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-eventtransform
  name: trigger-vm-add
spec:
  broker: broker-eventtransform
  filter:
    attributes:
      type: dev.knative.apiserver.resource.add
  subscriber:
    ref:
      apiVersion:  serving.knative.dev/v1
      kind: Service
      name: kn-py-psql-vmdata-fn
  delivery:
    retry: 1
    backoffPolicy: linear
    backoffDelay: PT5S
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  labels:
    eventing.knative.dev/broker: broker-eventtransform
  name: trigger-vm-delete
spec:
  broker: broker-eventtransform
  filter:
    attributes:
      type: dev.knative.apiserver.resource.delete
  subscriber:
    ref:
      apiVersion:  serving.knative.dev/v1
      kind: Service
      name: kn-py-psql-vmdata-fn
  delivery:
    retry: 1
    backoffPolicy: linear
    backoffDelay: PT5S
EOF
```


{{< image src="/img/posts//" caption="Figure I: " src-s="/img/posts//" >}}


{{< admonition info "" true >}}

{{< /admonition >}}

