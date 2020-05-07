# Event-driven interactions with vSphere using Functions as a Service

{{< admonition note "Note" true >}}
I am very happy to inform you that the below described idea has become a VMware Fling (completely OpenSource).

Official website of VMware Event Broker Appliance: https://vmweventbroker.io/
{{< /admonition >}}

When was the last time that you were so much taken by a topic or better by a combination of several, that just by thinking about it your face is filled with enthusiasm?

For me, it's only been a few days ago and it was in one of these afterwork-calls with my well appreciated colleague and friend <a href="https://mgasch.com" target="_blank">Michael Gasch</a> where we talked about things we were working on. I had just published my recent post on <a href="https://rguske.github.io/post/monitoring-container-vms-with-vrealize-operations-manager/" target="_blank">*"Monitoring Container VMs with vRealize Operations Manager"*</a> were I described how I built a dashboard for & with vRealize Operations Manager to monitor the utilization and performance for two of the major objects from vSphere Integrated Containers, the Virtual Container Host (VCH) and the Container-VMs (cVM).

Michael told me that he had spent some time in digging deeper and deeper into the topic **Functions as a Service**...

<center> {{< tweet 1055452330064334848 >}} </center>

which was far away from being experienced by myself, but he got me right away from the very beginning of his telling. For those of you who want to know more about the topic but haven't put your nose into it yet, like me before, you´ll find some links at the *Resources section* at the buttom.

I´m not going to explain what a Function/ FaaS is all about, instead of it Michael and I thought that this post should serve as an introduction of a project Michael is working on as well as to demonstrate how "just" two components are turning your vSphere environment into a *Event-driven construct* based on your *annotations*.

Before I´m going to start with the details and what should already be up and running, I`d like to quote the following description to explain what it means when it comes to the topic *"Functions vs. Containers"*.

{{< admonition info "Info" true >}}
**Container:**

Create a container which has all the required (Application) dependencies pre-installed, put your application code inside of it and run it everywhere the container runtime is installed.

**FaaS:**

Applications get split up into different functionalities (or services), which are in turn triggered by events. You upload your function code and attach an event source to it.

**Source:** <a href="https://serverless.com/blog/serverless-faas-vs-containers/" target="_blank">Serverless (FaaS) vs. Containers - when to pick which?</a>
{{< /admonition >}}

## Tagging vSphere objects by using a ƒ(x)

At the beginning I´ve mentioned that my last post was on the topic *"Monitoring Container VMs with vRealize Operations Manager"* and that I had to built a dashboard to do so. I´m using a widget in this dashboard called *Object List* to have the ability to select a specific Virtual Container Host, the **Resource Pool** not the Docker end-point VM(!), to display the utilization in terms of Limits & Reservations (CPU & Memory) of it.

While I was building this dashboard, one challange for me was, to find a way on which criteria these Resource-Pools will join a <a href="https://docs.vmware.com/en/vRealize-Operations-Manager/7.0/com.vmware.vcom.core.doc/GUID-7CD02B2B-4BB1-4BD3-B24E-5F630B94A7EF.html" target="_blank">Dynamic Custom Group</a> in vRealize Operations Manager (to appear automatically in the dashboard/ in the object list). Long story short, I decided to use vSphere Tags as a Membership-Criteria. But assigning a vSphere Tag is still a manually task (*1) and this is the point were Michaels function ***pytagfn*** or ***gotagfn***, depending on the language you prefer, as well as the ***vCenter Connector*** comes into play.

{{< admonition info "*1" true >}}
A Feature-Request for using the `vic-machine` utility to assign a vSphere-Tag during the VCH deployment is already open on Github:
https://github.com/vmware/vic/issues/8446
{{< /admonition >}}

The following flowchart will give you a high-level overview what it´s all about and what Michael and I will demo you in the Video below.

{{< image src="/img/posts/201902_vspheretagfn/CapturFiles-20190227_052627.jpg" caption="Figure I: Flowchart vSphere Tag assignment" src-s="/img/posts/201902_vspheretagfn/CapturFiles-20190227_052627.jpg" class="center" >}}

1. **vCenter-Connector** is listening to the event-stream coming from the vCenter Server and talks to the OpenFaaS-Gateway to retrieve a list of functions which are interested in incoming events.

2. `pytag-fn` will get invoked by a specific event (e.g. resource.pool.created).

3. **vSphere Tag** assignment through the triggered function.

4. **vRealize Operations Manager** is polling data from vCenter Server in certain intervals.

5. **Resource Pool** is **added** to the **Dynamic Custom Group** and is available in the dashboard/ object-list.

## The recording

{{< youtube X3usMlmyEoA >}}

## What you need | Pre-reqs

All you need is <a href="https://www.openfaas.com/" target="_blank">OpenFaaS</a> running in Kubernetes. As shown in the flowchart as well as having been mentioned in the recording, I´m running VMware´s Enterprise-grade Kubernetes solution <a href="https://cloud.vmware.com/vmware-pks" target="_blank">**VMware PKS**</a> in my <a href="https://twitter.com/search?q=%23homelab&src=typd" target="_blank">#Homelab</a> in order to make all necessary pieces available.

<center> {{< twitter 1089501596042649600 >}} </center>

Another pretty cool alternative way to run OpenFaaS on k8s is <a href="https://github.com/kubernetes-sigs/kind" target="_blank">KinD</a> which is a new tool from the Kubernetes community named **Kubernetes in Docker**.

{{< image src="/img/posts/201902_vspheretagfn/CapturFiles-20190228_092541.jpg" caption="Figure II: kind logo" src-s="/img/posts/201902_vspheretagfn/CapturFiles-20190228_092541.jpg" class="center" >}}

<a href="https://twitter.com/alexellisuk" target="_blank">Alex Ellis</a> has already written a nice blog-post on it: <a href="https://blog.alexellis.io/get-started-with-openfaas-and-kind/" target="_blank">*"Get started with OpenFaaS and KinD"*</a>. Check it out!

## Install OpenFaaS on Kubernetes

Installing OpenFaaS on Kubernetes is quite easy by using the "OpenFaaS-on-Kubernetes" Edition ***faas-netes***. Go through the documentation and you´ll have it installed in a couple of steps.

* <a href="https://github.com/openfaas/faas-netes/tree/master/chart/openfaas" target="_blank">OpenFaaS - Serverless Functions Made Simple</a>

* <a href="https://docs.openfaas.com/deployment/kubernetes/" target="_blank">OpenFaaS Docs - Deployment guide for Kubernetes</a>

* <a href="https://github.com/openfaas/faas-netes" target="_blank">Github Repository - faas-netes</a>

Having followed the instructions in the documentation, you will end up with the following components installed:

1. <a href="https://helm.sh/docs/install/" target="_blank">Helm</a> with the Helm
client (`helm`) and the Helm server (Tiller - running in the namespace kube-system).

2. OpenFaaS on Kubernetes
* deployed into the namespace *openfaas*

3. The faas-cli via e.g. `brew` or `curl`

4. <a href="https://github.com/openfaas-incubator/vcenter-connector" target="_blank">Github Repo `vCenter Connector`</a>

{{< admonition info "As of writing this post:" true >}}
The demo is based on a pending pull request (PR) in the vcenter-connector Github repository, which is currently under review. The PR, including all the details and files changed, can be found here: https://github.com/openfaas-incubator/vcenter-connector/pull/11
{{< /admonition >}}

5. <a href="https://github.com/embano1/pytagfn" target="_blank">Github Repo `pytag-fn`</a> or <a href="https://github.com/embano1/gotagfn" target="_blank">Github Repo `gotag-fn`</a>

I hope you enjoyed reading and liked the recording.

---

**<center>Thanks for reading</center>**

---

## Resources

**Article**   | **Link**
:-------------  | :-------------:
[@openfaaS] **Deployment guide for Kubernetes** | https://docs.openfaas.com/deployment/kubernetes/
[@github/openfaas-incubator] **Github repo vcenter-connector** | https://github.com/openfaas-incubator/vcenter-connector
[github/embano1] **Github repo pytagfn** | https://github.com/embano1/pytagfn
[github/embano1] **Github repo gotagfn** | https://github.com/embano1/pytagfn
[@blog.alexellis.io] **Get started with OpenFaaS and KinD** | https://blog.alexellis.io/get-started-with-openfaas-and-kind/
[@mgasch.com] **Events, the DNA of Kubernetes** | https://www.mgasch.com/post/k8sevents/
[@virtuallyghetto.com] **Getting started with VMware Pivotal Container Service (PKS) Part 1: Overview** | https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-1-overview.html
[@keithlee.ie] **PKS NSX-T Home Lab – Part 10: Install Ops Man and BOSH** | http://keithlee.ie/2018/12/09/pks-nsx-t-home-lab-part-10-install-ops-man-and-bosh/
[@beyondelastic.com] **VMware PKS 1.3 What’s New** | https://beyondelastic.com/2019/01/24/vmware-pks-1-3-whats-new/
[@blogs.vmware.com] **Dispatch to support Knative and Joins Cloud-Native Apps BU** | https://blogs.vmware.com/cloudnative/2018/10/25/dispatch-team-joins-cloud-native-apps-bu/
[@thehumblelab.com] **Getting Started with VMware Dispatch on Kubernetes** | https://www.thehumblelab.com/getting-started-with-vmware-dispatch-on-kubernetes/
[@serverless.com] **Serverless (FaaS) vs. Containers** | https://serverless.com/blog/serverless-faas-vs-containers/
[@medium.com] **Search query "Functions as a Service"** | https://medium.com/search?q=functions%20as%20a%20service
[@cloud.vmware.com] **VMware PKS Landing Page** | https://cloud.vmware.com/vmware-pks
[@pivotal.io] **Pivotal PKS Landing Page** | https://pivotal.io/platform/pivotal-container-service
