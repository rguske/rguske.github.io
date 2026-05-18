# KubeVirt meets Knative Eventing


## Introduction


```yaml
oc create -f - <<EOF
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: broker-apiserversource
spec: {}
---
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
  name: role-event-watcher
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
  name: event-display-raw
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
    eventing.knative.dev/broker: broker-apiserversource
  name: trigger-event-display-raw
spec:
  broker: broker-apiserversource
  filter: {}
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display-raw
EOF
```

```yaml
oc create -f - <<EOF
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
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  labels:
    eventing.knative.dev/broker: broker-eventtransform
  name: event-display-transformed
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
  name: trigger-event-display-transformed
spec:
  broker: broker-eventtransform
  filter: {}
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display-transformed
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

## Create a VirtualMachinePool

```yaml
oc create -f - <<EOF
apiVersion: pool.kubevirt.io/v1alpha1
kind: VirtualMachinePool
metadata:
  name: dev-fedora-vms
spec:
  replicas: 1
  selector:
    matchLabels:
      kubevirt.io/vmpool: dev-fedora-vms
  virtualMachineTemplate:
    metadata:
      labels:
        kubevirt.io/vmpool: dev-fedora-vms
    spec:
      dataVolumeTemplates:
      - metadata:
          name: fedora-dev-volume
        spec:
          sourceRef:
            kind: DataSource
            name: fedora
            namespace: openshift-virtualization-os-images
          storage:
            resources:
              requests:
                storage: 30Gi
            storageClassName: 'coe-netapp-san'
      runStrategy: Always
      template:
        metadata:
          labels:
            kubevirt.io/vmpool: dev-fedora-vms
        spec:
          domain:
            cpu:
              cores: 1
            memory:
              guest: 1024M
            devices:
              interfaces:
              - model: virtio
                name: coe-bridge-32
                state: up
                bridge: {}
          networks:
          - name: coe-bridge-32
            multus:
              networkName: "coe-bridge-32"
              disks:
              - disk:
                  bus: virtio
                name: rootdisk
              - disk:
                  bus: virtio
                name: cloudinitdisk
              rng: {}
          volumes:
          - dataVolume:
              name: fedora-dev-volume
            name: rootdisk
          - cloudInitNoCloud:
              userData: |-
                #cloud-config
                users:
                - name: rguske
                  gecos: R. Guske
                  groups: wheel
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  shell: /bin/bash
                  ssh_authorized_keys:
                    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINGf2ipOy2h3lJHbwP3qm4yICHvEFvYjlZGHWsEEUHCl jarvishomelab@gmail.com
            name: cloudinitdisk
EOF
```

`oc scale vmpool dev-eventing-vms --replicas 1`


## PostgreSQL DB StatefulSet


## The Function Deployment


```code
oc create secret generic psql-secret \
  --from-literal=db_host="10.32.98.110" \
  --from-literal=db_port="5432" \
  --from-literal=db_name="vmdb" \
  --from-literal=db_user="postgres" \
  --from-literal=db_password="redhat"
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

```code
oc -n rguske-eventing create secret generic pg-credentials \
  --from-literal=DB_HOST=10.32.98.110 \
  --from-literal=DB_USER=postgres \
  --from-literal=DB_PASSWORD='redhat'
```

```code
kn service create postgresql-read-webapp \
  --image=quay.io/rguske/psql-read-webapp:v1.1 \
  --env-from secret:pg-credentials \
  --env DB_NAME=vmdb \
  --env DB_PORT=5432 \
  --scale-min=0 \
  --scale-max=2
```


{{< image src="/img/posts//" caption="Figure I: " src-s="/img/posts//" >}}


{{< admonition info "" true >}}

{{< /admonition >}}

