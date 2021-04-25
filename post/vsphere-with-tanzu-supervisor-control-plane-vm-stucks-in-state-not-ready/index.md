# vSphere with Tanzu - SupervisorControlPlaneVM stucks in state NotReady

## Introduction

Power outage related circumstances recently brought my vSphere with Tanzu Workload Management[^1] enabled homelab cluster (Supervisor Cluster) into a not desired state which of course I had to fix.

And I do not mean that I have to protect my homelab against the next power outage by finally installing a UPS :wink:.

<center> {{< tweet 1384421366154285056 >}} </center>

What I actually mean is the following...

## Failed to get available workloads: bad gateway

The last couple of weeks I spent a lot of time using my Tanzu Kubernetes Cluster(s)[^2] to get my head around as well as my hands dirty on this awesome project Knative[^3]. More to come soon :wink:. Interacting with a healthy vSphere Supervisor Cluster[^4] is necessary for e.g. the provisioning of new Tanzu Kubernetes Cluster or the instantiation of vSphere Native Pods.

Unfortunately, my attempt to login into mine ends quicker than expected with the error message:

```shell
kubectl vsphere login --server=10.10.18.10 --vsphere-username administrator@jarvis.lab --insecure-skip-tls-verify

Password:
FATA[0004] Failed to get available workloads: bad gateway
Please contact your vSphere server administrator for assistance.
```

My first instinct was to quickly have a look at the Workload Management subsection in the vSphere Client (*Menu --> Workload Management or ctrl + alt + 7*) and here my suspicion that something is wrong was confirmed. The *Config Status* of my cluster *Malibu* was in `Configuring` state and the following two *Status Messages* as shown in *Figure I* were displayed.

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_092403.jpg" caption="Figure I: Supervisor Cluster stucks in configuring state" src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_092403.jpg" >}}

I observed this `Configuring` state for a while longer and I was hoping it gets fixed automagically but it didn't.

> Seriously, I do believe that this is really due to various (again) circumstances which I was facing with my homelab and normally the desired state (`ready`) for the Supervisor Cluster or more specifically, for the Supervisor Control Plane VMs, will be recovered automatically.

## Into troubleshooting

### HealthState WCP service unhealthy

The next move I did was checking the state of my Tanzu Kubernetes Cluster also via the vSphere Client but they were no longer displayed at all. Things went strange :ghost:.

Consequently, I checked the overall health of my vCenter Server as well as the health state of the services and especially my attention was on the `wcp` service (Workload Control Plane). Checking the service state can be done in two ways:

1. vCenter Appliance Management Interface aka VAMI (*vcenter url*:5480)
2. via the `shell` - which requires an enabled and running `ssh` service on the vCenter Server Appliance

The `shell` output was the following:


```shell
root@vcsa [ ~ ]# vmon-cli --status wcp
Name: wcp
Starttype: AUTOMATIC
RunState: STARTED
RunAsUser: root
CurrentRunStateDuration(ms): 72360341
HealthState: UNHEALTHY
FailStop: N/A
MainProcessId: 15679
```

> HealthState: UNHEALTHY

And here a little bit more coloured:

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_092113.jpg" caption="Figure II: vCenter Server service wcp state unhealthy " src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_092113.jpg" >}}

Let's see if the *HealthState* will change after restarting the service:

```shell
root@vcsa [ ~ ]# vmon-cli --restart wcp
Completed Restart service request.
root@vcsa [ ~ ]# vmon-cli --status wcp
Name: wcp
Starttype: AUTOMATIC
RunState: STARTED
RunAsUser: root
CurrentRunStateDuration(ms): 9709
HealthState: HEALTHY
FailStop: N/A
MainProcessId: 46317
```

> HealthState: HEALTHY

Well, way better. Let's move on from here.

### Supervisor Control Plane node status `NotReady`

Having the `wcp` service back in operating state led me to start over from where I began. This time logging in to my Supervisor Cluster went well and I checked the state of the three Control Plane Nodes by executing `kubectl get nodes`.


```shell
kubectl get nodes
NAME                               STATUS     ROLES    AGE     VERSION
42100ed03ae877fd39716f909d57822e   NotReady   master   7d19h   v1.19.1+wcp.2
4210a2e2fa945c1a13308fac6d00ed96   Ready      master   7d19h   v1.19.1+wcp.2
4210c09deaaa5a2e2ddc86d3e0c0b0f3   Ready      master   7d19h   v1.19.1+wcp.2
```

> Control Plane Node 1: STATUS NotReady

With this new gains, I also checked the Virtual Machine state via the Remote Console and surprisingly, it seems that the power outages affected this particular Node (VM) badly. The Remote Console window was swarmed with error messages saying `print_req_error: I/O error, dev sda, sector ...[counting up]`

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CaptureFiles-20-04-22.png" caption="Figure III: Control Plane VM status NotReady" src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CaptureFiles-20-04-22.png" >}}

I was also trying to figure out if there's a way to get this fixed on the Operating System level but there was no chance or at least there wasn't one for my expertise.

## vSphere ESXi Agent Manager

If the desired state cannot be recovered automatically again, what option remains?

Well, there's always something to learn. My well appreciated colleague [Dominik Zorgnotti](https://www.why-did-it.fail/) was pointing me to the [vSphere ESXi Agent Manager (EAM)](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.monitoring.doc/GUID-D56ABFF4-4529-409C-9AA2-8D8D4E235601.html), which in the end turned out to be the solution for my problem. To be honest, I never was in the situation to make use of the EAM before and therefore it wasn't on my radar at all but I was quite happy to get to know this component now. What it does?

{{< admonition quote "vSphere ESX Agent Manager" true >}}
The vSphere ESX Agent Manager automates the process of deploying and managing ESX and ESXi agents, which extend the function of the host to provide additional services to a vSphere solution.
{{< /admonition >}}

You will find the EAM in the vSphere Client under *Menu -> Administration -> vCenter Server Extensions -> vSphere ESXi Agent Manager*.

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_093111.jpg" caption="Figure IV: vSphere ESXi Agent Manager" src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_093111.jpg" >}}

The three Supervisor Control Plane VMs can be found via the *Configure* tab and are listed in the column as *Agency* named with prefix `vmware-vsc-apiserver-xxxxxx`. By selecting the three dots besides the name, it gives us the two options *Delete Agency* as well as *Remove All Issues*. See *Figure V*.

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095429.jpg" caption="Figure V: vSphere ESXi Agent Manager - List of Agency's" src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095429.jpg" >}}

But which of the three *"Agency's"* is our affected one? I've never recognized this used naming pattern before, not in the vSphere Client nor by using `kubectl` (e.g. `kubectl describe node`). I finally had a look at the summary page of the VM and the **Notes** widget enlightened me finally.

> This Virtual Machine is a VMware agent implementing support for vSphere Workloads. Its lifecycle operations are managed by VMware vCenter Server. EAM Agency: vmware-vsc-apiserver-w8mqd8
See also *Figure VI*:

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095346.jpg" caption="Figure VI: Agency name displayed on Notes widget" src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095346.jpg" >}}

Having found the missing piece, I first went with the *Remove All Issues* option but it didn't solve my problem.

With having in mind, that the Supervisor Cluster is a high available construct consisting of three members and it's current state is *degraded*, I checked twice if the one I picked is the right one before selecting the option *Delete Agency*.

Immediately after the selection, the affected VM got deleted and a new one was on it's way.

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095620.jpg" caption="Figure VII: Deployment of a new *SupervisorControlPlaneVM* " src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_095620.jpg" >}}

At the end, my cluster reached the *Config Status* `Running`, all three Nodes were `Ready` and the `wcp` service never became (until now) `unhealthy` again.

{{< image src="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_100748.jpg" caption="Figure I: " src-s="/img/posts/202104_v7k8s_supervisorvm_not_ready/CapturFiles-20210422_100748.jpg" >}}

```shell
kubectl get nodes
NAME                               STATUS   ROLES    AGE     VERSION
42103deb5d43d5f75f4623a336165684   Ready    master   3m37s   v1.19.1+wcp.2
4210a2e2fa945c1a13308fac6d00ed96   Ready    master   7d20h   v1.19.1+wcp.2
4210c09deaaa5a2e2ddc86d3e0c0b0f3   Ready    master   7d20h   v1.19.1+wcp.2
```

[^1]: [vSphere Workload Domain](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-152BE7D2-E227-4DAA-B527-557B564D9718.html)
[^2]: [What Is a Tanzu Kubernetes Cluster?](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-DC22EA6A-E086-4CFE-A7DA-2654891F5A12.html)
[^3]: [Knative Homepage](https://www.knative.dev)
[^4]: [Configuring and Managing a Supervisor Cluster](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-21ABC792-0A23-40EF-8D37-0367B483585E.html)

