# VMware vSphere Integrated Containers: Introduction


I´m very keen on everything related to Cloud-Native and all topics around the decoupling of a process from the underlying operating system and as well much more on the infrastructure underneath in order to run those distributed systems, which are the result by the end of the day.

For me personally, it became so present in 2016 for the first time and far more when I decided to work for VMware. Did you know that VMware is a huge contributor to Open Source Projects and a Platinum Member at the Cloud Native Computing Foundation - <a href="https://www.cncf.io/about/members/" target="_blank">CNCF</a>? You´ll find an overview of our open source projects <a href="https://vmware.github.io/" target="_blank"> here</a>.

One of these Open Source Projects is <a href="https://vmware.github.io/vic-product/" target="_blank">vSphere Integrated Containers</a> aka **VIC**. With vSphere Integrated Containers, Container Images get instantiated as a Virtual Machine by using <a href="https://vmware.github.io/photon/" target="_blank"> PhotonOS</a>, a minimal Linux Distribution by VMware, basically to be up and running in a few seconds.

Running Containers as a Virtual Machine means that IT-Ops Teams still have the ability to treat them like classical workload before. Therefore you don´t have to build out a separate, tailored infrastructure stack and can continue to leverage existing Scalability, Security and Monitoring capabilities.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180710_094308.jpg" caption="Figure I: Virtual Container Host vs. Docker Container Host" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180710_094308.jpg" class="center" width="800" height="800" >}}

vSphere Integrated Containers also supports running native Docker container hosts on vSphere. It allows developers to self-provision Docker container hosts for use as a development sandbox, a build server or a swarm cluster. Now you can treat a Docker host as ephemerally as a container.

In my role as a VMware Technical Account Manager I have the chance to meet individuals in various roles at Enterprises. Many share the same strategic priority: *Transform some of their monolithic applications.*

vSphere Integrated Containers can leverage a solution for use cases like <a href="https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/solutionbrief/vmware-vic-app-repackaging-use-case.pdf" target="_blank">App Repackaging</a> and <a href="https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/solutionbrief/vmware-vic-developer-sandbox-use-case.pdf" target="_blank">Developer Sandbox</a>. Just to name a few.

VMware vSphere Integrated Containers Version 1.4 was already launched by our Cloud-Native Business Unit (CNABU) on the 15th of March this year.
<center> {{< tweet 996453354812342274 >}} </center>

And while I´m writing this series...

<center> {{< tweet 1017665789258813440 >}} </center>

...VIC version 1.4.1. has been released and includes:

* Added the ability to manage use of DRS VM affinity groups for existing VCHs using vic-machine configure.

* When copying a CLI command on the Summary page of the VCH Creation Wizard, a confirmation message is now shown.

* New Built-In Repositories view in vSphere Integrated Containers Management Portal provides better visibility on all repositories that reside in the vSphere Integrated Containers Registry.

* Security and bug fixes.

More details through the Release-Notes <a href="https://docs.vmware.com/en/VMware-vSphere-Integrated-Containers/1.4.1/rn/vsphere-integrated-containers-141-release-notes.html" target="_blank">here</a>.

---

## VIC Component Overview

You´ll get an quick overview over the VIC components through this Lightboard Video by <a href="https://twitter.com/pdaigle" target="_blank">Patrick Daigle</a>:
{{< youtube phsVFTVK4t4 >}}

### Components:

* <a href="http://vmware.github.io/vic/" target="_blank">**vSphere Integrated Containers Engine**</a>, a container runtime for vSphere that allows you to provision containers as virtual machines.

* **vSphere Integrated Containers Plug-In for vSphere Client**, that provides information about your vSphere Integrated Containers setup and allows you to deploy virtual container hosts directly from the vSphere Client.

* <a href="https://vmware.github.io/harbor/" target="_blank">**Harbor**</a>, **an enterprise-class container registry server** that stores and distributes container images. 

 * Multi-tenant content signing and validation
 * Security and vulnerability analysis
 * Audit logging
 * Identity integration and role-based access control
 * Image replication between instances
 * Extensible API and graphical UI

* **vSphere Integrated Containers Management Portal** based on project <a href="https://github.com/vmware/admiral" target="_blank">**Admiral**</a>.

## Download
* <a href="https://www.vmware.com/go/download-vic" target="_blank">VMware vSphere Integrated Containers</a>

## More

### [VIC Solutions Brief]

* <a href="https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/products/vsphere/vmware-vic-solutions-brief.pdf" target="_blank">Run Modern Apps with vSphere Integrated Containers</a>

### Whitepapers

* <a href="https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/whitepaper/vmw-wp-conatiner-on-vms-a4-final-web.pdf" target="_blank">Containers on Virtual Machines or Bare Metal?</a>

* <a href="https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/vsphere/vmware-vsphere-integrated-containers-white-paper.pdf" target="_blank">Architecture Overview (VIC 1.2)</a>

### Documentation

* <a href="https://vmware.github.io/vic-product/#documentation" target="_blank">Official documentation page</a>

* <a href="https://github.com/vmware/vic" target="_blank"> VIC on Github</a>

* <a href="https://github.com/vmware/vic-ui" target="_blank">VIC UI on Github</a>

### Release Notes

* <a href="https://docs.vmware.com/en/VMware-vSphere-Integrated-Containers/" target="_blank">Release Notes vSphere Integrated Containers</a>

### VMware Hands-on Labs

* <a href="https://labs.hol.vmware.com/HOL/catalogs/lab/4277" target="_blank">HOL-1830-02-CNA - vSphere Integrated Containers – Getting Started</a>

### Offical VMware Cloud-Native Blog

* <a href="https://blogs.vmware.com/cloudnative" target="_blank">VMware Cloud Native Blog</a>
* <a href="https://blogs.vmware.com/cloudnative/?s=vsphere+integrated+containers" target="_blank">Blog search query - VIC</a>

### Recommended VMworld Videos

<a href="https://videos.vmworld.com/global/2017/videoplayer/3703" target="_blank">vSphere Integrated Containers Deep Dive: Cool Hacks, Debugging, and Demos (CNA2547BU)</a>
{{< youtube AD7CD8Haqdc >}}

<a href="https://videos.vmworld.com/global/2017/videoplayer/3238" target="_blank">Running Docker on Your Existing Infrastructure with vSphere Integrated Containers (CNA1699BU)</a>
{{< youtube d5nf8tn2MIM >}}

<a href="https://videos.vmworld.com/global/2018/videoplayer/22401" target="_blank">Deploy vSphere Integrated Containers in Production: Case Study with Allegis (CNA1200BU)</a>

### Lightboard Videos

<a href="https://www.youtube.com/playlist?list=PL7bmigfV0EqRxUB5FND_5tRdmM1qdC_Hl" target="_blank"> VMware Cloud-Native YouTube Channel - VIC Lightboard Videos Overview</a>
{{< youtube phsVFTVK4t4 >}}

<a href="https://www.youtube.com/watch?v=QLi9KasWLCM&t=1s&list=PL7bmigfV0EqRxUB5FND_5tRdmM1qdC_Hl&index=3" target="_blank">vSphere Integrated Containers: Networking Overview</a>
{{< youtube QLi9KasWLCM >}}

<a href="https://www.youtube.com/watch?v=WrVHbjTZHrs&t=0s&list=PL7bmigfV0EqRxUB5FND_5tRdmM1qdC_Hl&index=4" target="_blank">vSphere Integrated Containers: Storage</a>
{{< youtube WrVHbjTZHrs >}}

---

**Continue with:**

<a href="/post/vmware-vsphere-integrated-containers-part-1-ova-deployment/">**vSphere Integrated Containers Part I: OVA Deployment**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-2-vsphere-client-plugin/">**vSphere Integrated Containers Part II: vSphere Client Plug-In**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-3-deployment-of-a-virtual-container-host/">**vSphere Integrated Containers Part III: Deployment of a Virtual Container Host**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-4-docker-run-a-container-vm/">**vSphere Integrated Containers Part IV: docker run a Container-VM**</a>

---

**<center>Thanks for reading</center>**
