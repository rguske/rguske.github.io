# Leveraging Fluent Bit on Tanzu Kubernetes Grid to send VEBA logs to vRealize Log Insight

## Introduction

With the recent v0.5 release of the [VMware Event Broker Appliance (VEBA)](https://vmweventbroker.io) project, the ability of deploying the core component, the [VMware Event Router](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/vmware-event-router), via a Helm chart to an existing Kubernetes cluster was introduced. [William Lam](https://twitter.com/lamw) introduced this and all the other great new features and enhancements in his corresponding [blog post](https://www.virtuallyghetto.com/2020/12/vcenter-event-broker-appliance-veba-v0-5-0.html). This earned very positive feedback from the community and opens even more doors in terms of flexibility as well as extensibility.

Speaking of which, extensibility...

During the v0.5 Pre-Release team meeting, William came up with the idea to let VEBA sending logs to our log analysis solutions VMware [vRealize Log Insight](https://www.vmware.com/products/vrealize-log-insight.html) or [vRealize Log Insight Cloud](https://cloud.vmware.com/log-insight-cloud). He asked me if I would do it and of course I found it a great idea. This post is a good addition to my previous post *"Monitoring the VMware Event Broker Appliance with vRealize Operations Manager"*[^1], in which I'm describing how I used cAdvisor[^2] to get resource usage and performance data from the various VEBA components and ultimately to monitor them.

The Open Source projects [Fluentd](https://www.fluentd.org/) and [Fluent Bit](https://fluentbit.io/) can be leveraged for multi-platform log processing and log forwarding. For my use case with VEBA, running in a [Tanzu Kubernetes Grid - TKGs](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-70CAF0BB-1722-4526-9CE7-D5C92C15D7D0.html) cluster (also referred to as *Guestcluster*) on VMware vSphere, my requirements on the solution were basically simple. It should be lightweight, fast and if possible, with less or better without dependencies. Comparing both mentioned projects with each other, it brought me multiple times back to Fluent Bit, even besides the fact, that good content on how to integrate Fluentd with TKG already exist.

## The Hummingbird (Note: no technical relevance)

The power of symbols thrills me. It's that power that makes you think or remember a specific brand, a company or an event. Just think about the Kubernetes steering wheel for example. If you feel home in the cloud native space and you see a steering wheel in real life nowadays, you will probably think of Kubernetes. This is the tremendous power of symbols[^3]. That is why there are fields of study for Symbology and movies with famous actors who are playing characters like Robert Langdon :wink:.

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_055615.jpg" caption="Figure I: Project Logos Fluentd and Fluent Bit" width="500" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_055615.jpg" >}}

Now, if we look at the logos of both projects (*Figure I*), the maintainers did a great job in choosing a Carrier Pigeon[^4] for Fluentd and a Hummingbird[^5] for Fluent Bit. Fluentd supports an extensive library of over 1000 [^6] plugins to provide extended support and functionality for anything related to log and data management. A Carrier Pigeon is known for it's impressive performance e.g. in terms of distance covered...and it carries out data :smiley::bulb:.

{{< admonition note "BTW" true >}}
Fluentd  <i class='fab fa-github fa-fw'></i>-issue [#924](https://github.com/fluent/fluentd/issues/924) from 2016 was about "A Fluentd new logo proposal" and is an intersting short read, why a change was requested and why it was important.
{{< /admonition >}}

Hummingbirds in comparison to, are lightweight and PRETTY fast. They have wing-flapping rates between 12 and 80 beats per second (depending on their actual size). Fluent Bit is designed to perform fast and efficient with least amount of resource usage.

*From the official Fluent Bit homepage:*

{{< admonition quote "" true >}}
Fluent Bit is designed with performance in mind: high throughput with low CPU and Memory usage.
{{< /admonition >}}

## A short comparison

Both projects are very strong and feature-rich and great articles already exist on "how to deploy" Fluentd to forward logs to e.g. vRealize Log Insight Cloud. I put some of them in the resource section below. The following table compares both projects in specific areas.

| | **Fluentd** | **Fluent Bit** |
|:---: |:---: |:---: |
| Scope | Containers / Servers | Embedded Linux / Containers / Servers |
| Language | C & Ruby | C |
| Memory | ~40MB | ~650KB |
| Performance | High Performance | High Performance |
| Dependencies | Built as a Ruby Gem, it requires a certain number of gems. | Zero dependencies, unless some special plugin requires them |
| Plugins | More than 1000 plugins available | Around 70 plugins available |
| License | Apache License v2.0 | Apache License v2.0 |

*Source: [Fluentd & Fluent Bit](https://docs.fluentbit.io/manual/about/fluentd-and-fluent-bit)*

## Logging Primitives

Before we start deploying and leveraging Fluent Bit to read and forward logs to VMware's vRealize Log Insight, I wanted to briefly explain or refer you to some of the logging primitives of Cloud Native Applications or Services itself as well as how Fluent Bit is and can be configured. I found this good article during my research - *[Logging from Docker Containers to Elasticsearch with Fluent Bit](https://kevcodez.de/posts/2019-08-10-fluent-bit-docker-logging-driver-elasticsearch/)* -  in which the author quoted the following statement from the Twelve-Factor App[^7] website regarding `Logs`.

{{< admonition quote "https://12factor.net/logs" true >}}
*A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to stdout. During local development, the developer will view this stream in the foreground of their terminal to observe the app’s behavior.*
{{< /admonition >}}

Doesn't it make absolutely sense? Fluent Bit offers several `Input` plugins to collect logs and data from sources. From a high-level perspective, the structure of Fluent Bit looks as follows :

{{< mermaid >}}
graph LR;
    L[ Source]
    subgraph Inputs
    A[ Standard Input ]
    B[ Systemd ]
    C[ Tail ]
    D[ ... ]
    end
    E[ Fluent Bit ]
    subgraph Outputs
    F[ Standard Output ]
    G[ Syslog ]
    H[ Kafka ]
    J[ ... ]
    end
    A & B & C --- E
    E --- F & G & H
    K[ Destination]
    F & G & H --> K
    L --- A & B & C
{{< /mermaid >}}

To structure the collected data, `Parsers` can be used to convert them into a structured format. Also, `Filters` are configurable to e.g. append additional information or to filter out unwanted log statements. Links to even more details can also be found in the resource section.

## Prerequisites

To get Fluent Bit in working state on your Kubernetes cluster, we do need a `ServiceAccount` with an assigned `ClusterRole` as well as with it's appropriate `ClusterRoleBinding`. We will deploy Fluent Bit as a Kubernetes DaemonSet[^8] in a `Namespace` called `fluent-bit`. I've created a repository on <i class='fab fa-github fa-fw'></i>Github which you can `clone`, make the appropriate adjustments to the files and apply everything easily onto your cluster. It contains the following four files which sequence of deplyoment can be identified based on the number at the beginning of the filename (except #4).

- `1-tkg-fluent-bit-preps.yaml`
- `2-tkg-fluent-bit-configmap-cri.yaml`
- `3-tkg-fluent-bit-ds.yaml`
- `4-tkg-fluent-bit-configmap-docker.yaml`

### Containerd as Runtime - CRI Parser used!

As aforementioned, I'm using a TKG cluster for my example respectively in my environment. VMware is using Containerd[^9] as it's container runtime and therefore I had to adjust my configuration accordingly. If you are using Docker as your runtime, simply use the file `4-tkg-fluent-bit-configmap-docker.yaml` as an alternative.

Even though, for both applies:

{{< admonition quote "" true >}}
*By default, container engines such as Docker capture the standard output or error and leverage the JSON-file driver on each host to write messages to files. Docker maintains a separate log file for each container and stores it in the /var/log/containers directory of the Docker host. Annotation for each log entry consists of the following:*

- Log message
- Message origin - stdout or stderr
- Timestamp

*Source: [The Big Easy: Visualizing Logging Data by Integrating Fluentd and vRealize Log Insight with VMware PKS](https://tanzu.vmware.com/content/blog/the-big-easy-visualizing-logging-data-by-integrating-fluentd-and-vrealize-log-insight-with-vmware-pks)*
{{< /admonition >}}

### <i class='fab fa-github fa-fw'></i> Repository

1. Clone the repository locally and change into the directory `fluent-bit-vmware-loginsight`: `git clone https://github.com/rguske/fluent-bit-vmware-loginsight.git && cd fluent-bit-vmware-loginsight`

### Adjustments

2. Make your adjustments to the following files to meet your requirements and environment specifications:

- `2-tkg-fluent-bit-configmap-cri.yaml`
- `3-tkg-fluent-bit-ds.yaml`

#### Fluent Bit Configmap

The first adjustment you have to make is the replacement of your TKG clustername as well as the hostname of your vRealize Log Insight appliance.

File `2-tkg-fluent-bit-configmap-cri.yaml`:

- Replace the data for your TKG cluster in the `filter-record.conf: |` section
  - Note: for non TKG cluster, just delete the record

`filter-record.conf: |`

```code
[FILTER]
    Name                record_modifier
    Match               *
    Record tkg_instance tkg-veba
    Record tkg_cluster  tkg-veba
```

Enter your Log Insight server hostname in the `output-syslog.conf` section:

`output-syslog.conf: |`

```code
[OUTPUT]
    Name                 syslog
    Match                *
    Host                 vrli.jarvis.lab
    Port                 514
    Mode                 tcp
    Syslog_Format        rfc5424
    Syslog_Hostname_key  tkg_cluster
    Syslog_Appname_key   pod_name
    Syslog_Procid_key    container_name
    Syslog_Message_key   message
    syslog_msgid_key     msgid
    Syslog_SD_key        k8s
    Syslog_SD_key        labels
    Syslog_SD_key        annotations
    Syslog_SD_key        tkg
```

#### Fluent Bit DaemonSet

Basically, it's not necessary to change the image for Fluent Bit but you could replace it with the one of your choice. I used the official Fluent Bit image which is at the time of writing this post version 1.6.9 -  `fluent/fluent-bit:1.6.9`. You could also use e.g. the VMware Fluent Bit Image in version 1.5.3: `registry.tkg.vmware.run/fluent-bit:v1.5.3_vmware.1`. I've validated the functionality of both images during my tests.

More images:

- Official: https://hub.docker.com/r/fluent/fluent-bit
- Bitnami: https://hub.docker.com/r/bitnami/fluent-bit
- VMware Registry: registry.tkg.vmware.run/fluent-bit:v1.5.3_vmware.1

File `3-tkg-fluent-bit-ds.yaml`:

- Make your edits at the `spec` section for the container:

```yaml
[...]

spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:1.6.9

[...]
```

## Deploy Fluent Bit on Kubernetes

3. Start with applying the `1-tkg-fluent-bit-preps.yaml` file to have the prerequisites (`Namespace`, `ServiceAccount`, etc.) done:

```shell
$ kubectl create -f 1-tkg-fluent-bit-preps.yaml

namespace/fluent-bit created
serviceaccount/fluent-bit created
clusterrole.rbac.authorization.k8s.io/fluent-bit-read created
clusterrolebinding.rbac.authorization.k8s.io/fluent-bit-read created
```

4. Next is to apply the configmap - `2-tkg-fluent-bit-configmap-cri.yaml` - for Fluent Bit:

```shell
kubectl create -f 2-tkg-fluent-bit-configmap-cri.yaml
configmap/fluent-bit-config created
```

1. As mentioned in the Prerequisites section before, Fluent Bit will be deployed as a DaemonSet by using the `3-tkg-fluent-bit-ds.yaml` file:

```shell
kubectl create -f 3-tkg-fluent-bit-ds.yaml && kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n fluent-bit
daemonset.apps/fluent-bit created
pod/fluent-bit-dd4ql condition met

kubectl get daemonsets.apps -n fluent-bit

NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluent-bit   2         2         2       2            2           <none>          95s

kubectl get pods -n fluent-bit
NAME               READY   STATUS    RESTARTS   AGE
fluent-bit-9m7qq   1/1     Running   0          108s
fluent-bit-dd4ql   1/1     Running   0          108s
```

More information about the successful deployment and as a first indication that everything is working fine can be grabbed from the logs of one of the pods.

```code
kubectl logs -n fluent-bit fluent-bit-dd4ql

Fluent Bit v1.6.9
* Copyright (C) 2019-2020 The Fluent Bit Authors
* Copyright (C) 2015-2018 Treasure Data
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2021/01/12 14:05:10] [ info] [engine] started (pid=1)
[2021/01/12 14:05:10] [ info] [storage] version=1.0.6, initializing...
[2021/01/12 14:05:10] [ info] [storage] in-memory
[2021/01/12 14:05:10] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
[2021/01/12 14:05:11] [ info] [input:systemd:systemd.1] seek_cursor=s=f34691953f354fc19dbd06139503c160;i=156... OK
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] https=1 host=kubernetes.default.svc port=443
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] local POD info OK
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] testing connectivity with API server...
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] API server connectivity OK
[2021/01/12 14:05:11] [ info] [output:syslog:syslog.0] setup done for vrli.jarvis.lab:514
[2021/01/12 14:05:11] [ info] [http_server] listen iface=0.0.0.0 tcp_port=2020
[2021/01/12 14:05:11] [ info] [sp] stream processor started
```

## What vRealize Log Insight shows

The log output from the Fluent Bit pods itself looks promising. Now let's check what vRLI has to offer. Login and check the `Hosts` section under `Administration`:

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_115153.jpg" caption="Figure II: TKG cluster is listed in vRLI" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_115153.jpg" >}}

My TKG cluster `tkg-veba` is showing up - nice!

---

{{< admonition warning "" true >}}
During my tests, I deleted the `filter-record` value `tkg_cluster` at some point in the Configmap which resulted in the following misbehavior - *Figure III*.
{{< /admonition >}}

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_112805.jpg" caption="Figure III: vRLI Dashboard for Function Invocation Count" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210111_112805.jpg" >}}

The cluster hostname is not showing up anymore - so this isn't applicable.

---

Now switching to `Interactive Analytics` to finally see what Fluent Bit is providing us and if our configuration is working as expected. I've filtered for `hostname contains tkg-veba`:

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_035514.jpg" caption="Figure IV : vRLI Interactive Analytics" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_035514.jpg" >}}

## Monitoring Function Invocation Count

Coming to the end of this post, I've created a simple Dashboard which is showing the invocation count of two functions. Both functions were invoked on a `DrsVmPoweredOn` and `DrsVmPoweredOff` event. This is how the log output looks like for e.g. the [PowerCLI tagging function](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/master/examples/powercli/tagging) and the [Python restpost-fn](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/master/examples/python/invoke-rest-api).

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_095239.jpg" caption="Figure V: vRLI Dashboard Filters" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_095239.jpg" >}}

*Figure VI* below is showing the dashboard and the invocation count of both functions based on the incoming logs delivered by Fluent Bit.

{{< image src="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_032027.jpg" caption="Figure VI: vRLI Dashboard for Function Invocation Count" src-s="/img/posts/202101_veba_fluentbit_vrli/CapturFiles-20210112_032027.jpg" >}}

Stay healthy and stay safe!

[^1]: [Monitoring the VMware Event Broker Appliance with vRealize Operations Manager](https://rguske.github.io/post/monitoring-the-vmware-event-broker-appliance-with-vrealize-operations-manager/)
[^2]: [cAdvisor](https://github.com/google/cadvisor)
[^3]: [Symbols](https://en.wikipedia.org/wiki/Symbol)
[^4]: [Carrier Pigeon](https://en.wikipedia.org/wiki/Homing_pigeon)
[^5]: [Hummingbird](https://en.wikipedia.org/wiki/Hummingbird)
[^6]: [Fluentd - List of All Plugins](https://www.fluentd.org/plugins/all)
[^7]: [Twelve-Factor App](https://12factor.net/)
[^8]: [Kubernetes DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
[^9]: [Containerd Homepage](https://containerd.io/)

## Resources

- [Fluentd Log Insight Plugin](https://github.com/vmware/fluent-plugin-vmware-loginsight)
- [Configure log forwarding from Tanzu Kubernetes Grid (TKG) to vRealize Log Insight Cloud](https://www.virtuallyghetto.com/2020/04/configure-log-forwarding-from-tanzu-kubernetes-grid-tkg-to-vrealize-log-insight-cloud.html)
- [Introducing vRealize LogInsight Cloud Helm Chart for Kubernetes Logs](https://vcommunique.blogspot.com/2020/11/introducing-vrealize-loginsight-cloud.html)
- [Implementing Log Forwarding with Fluent Bit](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.2/vmware-tanzu-kubernetes-grid-12/GUID-extensions-logging-fluentbit.html#http)
- [Fluent Bit - Configuration File/ Configmap](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file)
- [Fluent Bit - Input Plugins](https://docs.fluentbit.io/manual/pipeline/inputs)
- [Fluent Bit - Input Plugins Github](https://github.com/fluent/fluent-bit/#input-plugins)
- [Fluent Bit - Output Plugins](https://docs.fluentbit.io/manual/pipeline/outputs/)
- [Fluent Bit Log Level](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file)
- [vRealize Log Insight - Ports](https://docs.vmware.com/en/vRealize-Log-Insight/8.2/com.vmware.log-insight.administration.doc/GUID-14DBC90A-379A-4316-9D76-4850E08437A8.html)
- [Supported RFC's LogInsight](https://docs.vmware.com/en/vRealize-Log-Insight/8.2/log-insight-administration-guide.pdf)
- [CRI Website](https://rubular.com/r/tjUt3Awgg4)
