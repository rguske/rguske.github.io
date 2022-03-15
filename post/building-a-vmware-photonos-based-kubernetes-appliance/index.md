# Building a VMware PhotonOS based Kubernetes Appliance for vSphere


## Why a Kubernetes-Appliance?

Learning Kubernetes (K8s) is still in great demand and is by no means decreasing. Starting with the theory is essential but we all know, the fun part starts with pactice. Quickly instantiating a Kubernetes cluster is made easy and I'm pretty sure it won't take long until [KinD](https://github.com/kubernetes-sigs/kind) crosses your path. But sometimes, a local K8s instance isn't enough and practicing Kubernetes low-level implementations like e.g. for networking ([CNI](https://kubernetes.io/docs/concepts/cluster-administration/networking/)) or storage ([CSI](https://kubernetes-csi.github.io/docs/)) on real cloud environments is crucial.

If you are with vSphere like me, you surely appreciate the simplicity of the [VMware Open Virtualization Appliance (OVA)](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-AFEDC48B-C96F-4088-9C1F-4F0A30E965DE.html) format. Not only the majority of VMware solutions are provided in this format, it's also pretty popular to build and provide VMware [Flings](https://flings.vmware.com/) in this format too.

Like the VMware open-source project [VMware Event Broker Appliance - VEBA](https://vmweventbroker.io/) for instance.

Thus, why not developing an easy deployable Kubernetes-Appliance which you can quickly deploy on vSphere for e.g. testing, development or learning purposes.

## Kubernetes-Appliance Project on Github

As being part of the VMware internal team around the VEBA project, I had the early opportunity to dive deeper into the build process of such an appliance, which was greatly invented by [William Lam](https://twitter.com/lamw) and [Michael Gasch](https://twitter.com/embano1). So, I took the code basis of the [v0.6.0](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/release-0.6.0) release, stripped it down to it's essential pieces, adjusted it accordingly and added new code to it, so it met my requirements.

---
All details on how-to build the appliance by your own as well as a download link are available on the project homepage on Github.

- <i class='fab fa-github fa-fw'></i> [rguske/kubernetes-appliance](https://github.com/rguske/kubernetes-appliance) <i class='fab fa-github fa-fw'></i>

---

### Kubernetes-Appliance Core Components

The appliance is build on the following core components:

| Operating System | CNI | CSI |
| :--: | :--: | :--: |
| [VMware PhotonOS](https://vmware.github.io/photon/) | [Antrea](https://antrea.io/) | [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner) |
| Open-source minimalist Linux operating system that is optimized for cloud computing platforms, VMware vSphere deployments, and applications native to the cloud. | A Kubernetes-native project that implements the Container Network Interface (CNI) and Kubernetes NetworkPolicy thereby providing network connectivity and security for pod workloads.  | Provides a way for the Kubernetes users to utilize the local storage in each node. |

### Additional Installations/Goodies

When you are getting more and more into Kubernetes, and your time on the shell is increasing heavily, it's a pretty good idea to start enhancing your shell experience by making use of additional CLI's, tools, shell extensions as well as themes to make your lives easier and more colorful. Therefore, after deploying the appliance and connecting to it via `ssh`, you will get to use:

- [zsh](http://www.zsh.org/)
- [oh-my-zsh](https://ohmyz.sh/)
- [Powerlevel9k](https://github.com/Powerlevel9k/powerlevel9k)
- [Helm](https://helm.sh/)

Details on the ZSH configuration here: [setup-05-shell.sh](https://github.com/rguske/kubernetes-appliance/blob/main/files/setup-05-shell.sh)

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-8.png" caption="Figure I: ZSH Shell Environnment" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-8.png" >}}

## Deployment Options

The following options will be available when deploying the appliance onto vSphere.

- **Review Details**

Compare the displayed version against the [release-versions](https://github.com/rguske/kubernetes-appliance/releases) on Github to ensure running latest.

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-1.png" caption="Figure II: OVA - Review Template Details" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-1.png" >}}

- **Networking**

Enter the necessary data for the appliance networking configuration.
{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-2.png" caption="Figure III: OVA - Networking Configuration" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-2.png" >}}

### Feature: Join an existing K8s-Appliance

Since the Kubernetes part of the appliance is installed using `kubeadm`, I thought making an option available to join an existing K8s-Appliance node while running through the OVA deployment sections, could be a cool feature.

Details: [setup-04-kubernetes.sh](https://github.com/rguske/kubernetes-appliance/blob/main/files/setup-04-kubernetes.sh)

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-3.png" caption="Figure IV: OVA - Joining an existing K8s-Appliance Node Option" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-3.png" >}}

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-7.png" caption="Figure V: OVA - Joining an existing K8s-Appliance Node" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-6.png" >}}

- **Proxy Configuration**

The appliance deployment supports the configuration of a Proxy. The configuration applies on the OS level as well as for Docker.

See here: [setup-02-proxy.sh](https://github.com/rguske/kubernetes-appliance/blob/main/files/setup-02-proxy.sh)

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-4.png" caption="Figure VI: OVA - Proxy Configuration" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-4.png" >}}

- **OS Credentials and Advanced**

Specify the password for the root user as well as choose if you want to enable the debugging ([/var/log/bootstrap-debug.log](https://github.com/rguske/kubernetes-appliance/blob/30b3e6499cb2c1ce86b117d65ad9f15bed2db9ff/files/setup.sh#L32)) log.

Finally, if necessary adjust the CIDR's for Docker Brigdge Networking and/or for the Kubernetes PODs.

{{< image src="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-5.png" caption="Figure VII: OVA - OS Configuration" src-s="/img/posts/202203_kubernetes_appliance/rguske-k8s-app-ova-5.png" >}}


That's it. I hope the appliance will serve you in the way I thought it could do :smile:

<center> {{< tweet user="vmw_rguske" id="1501959909964984322" >}} </center>

