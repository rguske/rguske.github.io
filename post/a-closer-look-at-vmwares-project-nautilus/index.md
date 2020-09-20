# A closer look at VMware's Project Nautilus

## General Availability Fusion 12 & Workstation 16
VMware's Desktop Hypervisor solutions [Fusion](https://www.vmware.com/products/fusion/faq.html) for Mac users and [Workstation](https://www.vmware.com/products/workstation-pro/faq.html) for the Windows and Linux userbase were launched in it's newest versions on September 15th. Lots of new features, enhancements and support around VM Guest Operating Systems, VM Scaling, GPU, Containers and Kubernetes has made it into these releases. The made enhancements will enrich Developers tool kit as well as will provide great new capabilities for IT Admins and everyone else who is keen on spinning up and down Virtual Machines, Containers and **NOW** also Kubernetes.

## A short recap on Project Nautilus
With Fusion Pro Tech Preview 20H1, VMware introduced [Project Nautilus](https://blogs.vmware.com/teamfusion/2020/01/fusion-tp20h1-introducing-nautilus.html) earlier this year (January). The main goal of the project is to provide a single development platform on the desktop by enabling users to run [OCI](https://opencontainers.org/) compliant **Containers** as well as **Kubernetes** besides **Virtual Machines** on the desktop. A couple of months later (May), Fusion version 11.5 went GA and with this, the possibility was given to manage containers (build & run) as well as VMware's [containerd](https://containerd.io/) based runtime. Checkout the corresponding blog post to get to know more about it: :arrow_right: [Fusion 11.5.5 Available Now](https://blogs.vmware.com/teamfusion/2020/05/fusion-11-5-now-supports-containers.html).

In order to make use of the intruduced "Container feature", a new command-line utility named `vctl` is automatically installed when installing Fusion version 11.5 and higher or Workstation 16 (depending on your OS). Users who are familiar with Docker will propably get a confidential feeling very quickly when using it. Let me show you what I mean by executing `vctl`in my terminal:

```
vctl

vctl - A CLI tool for the container engine powered by VMware Fusion
vctl Highlights:
• Build and run OCI containers.
• Push and pull container images between remote registries & local storage.
• Use a lightweight virtual machine (CRX VM) based on VMware Photon OS to host a container. Use 'vctl system config -h' to learn more.
• Easy shell access into virtual machine that hosts container. See 'vctl execvm’.

USAGE:
  vctl COMMAND [OPTIONS]

COMMANDS:
  build       Build a container image from a Dockerfile.
  create      Create a new container from a container image.
  describe    Show details of a container.
  exec        Execute a command within a running container.
  execvm      Execute a command within a running virtual machine that hosts container.
  help        Help about any command.
  images      List container images.
  ps          List containers.
  pull        Pull a container image from a registry.
  push        Push a container image to a registry.
  rm          Remove one or more containers.
  rmi         Remove one or more container images.
  run         Run a new container from a container image.
  start       Start an existing container.
  stop        Stop a container.
  system      Manage the container engine.
  tag         Tag container images.
  version     Print the version of vctl.

Run 'vctl COMMAND --help' for more information on a command.

OPTIONS:
  -h, --help   Help for vctl
```

What you see are pretty basic capabilities to **build**, **run** and **manage** containers locally with `vctl`.

To get started with `vctl`, I'd like to point you directly to the official `vctl` [Getting Started Guide](https://github.com/VMwareFusion/vctl-docs/blob/master/docs/getting-started.md) on <i class='fab fa-github fa-fw'></i> Github.

## New versions bring a KinD way to deploy Kubernetes Clusters
As mentioned in the previous section of this post, it has been planned from the very beginning of Project Nautilus to provide both, the ability to run containers AND to instantiate Kubernetes clusters on the desktop. This has now been made possible by making use of the open source project [KinD](https://kind.sigs.k8s.io/) (Kubernetes in Docker), which is using Docker container "nodes" to run local Kubernetes clusters. KinD has reached a decent popularity in the community and it can be easily installed on various platforms by e.g. using Brew for Mac, Chocolatey for Windows and `curl` in general. See also the [KinD Quick Start guide](https://kind.sigs.k8s.io/docs/user/quick-start).

So, what's in for you when using Fusion or Workstation instead of the "known, step-by-step, manual way"?

Advantages I see are e.g. not having the binaries of `docker`, `kind` and `kubectl` installed in advance on your platform as well as not to carry about the installation process/ method(s) itself. But let me show you some of the implementation details now, to hopefully enlighten you more. `vctl` is available after the installation of Fusion or Workstation.

**One thing before I go on!** I will not cover both solutions! I'm a Mac user, so from here on I'm going to concentrate just on Fusion 12.

### Let's dive in

First! The `~/.vctl` directory doesn't exist before you have executed `vctl system start` for the first time!

```
cd ~/.vctl
cd: no such file or directory: /Users/rguske/.vctl
```

Run `vctl system start` to start the containerd runtime daemon in the background and to have the aforementioned directory created.

```
vctl system start
Preparing storage...
Container storage has been prepared successfully under /Users/rguske/.vctl/storage
Launching container runtime...
Container runtime has been started.
```

How does it look now?

```
~/.vctl
tree -L 2
.
├── Fusion\ Container\ Storage.sparseimage
├── config.json
├── config.toml
├── config.yaml
├── containerd.log
├── opt
│   └── containerd
└── storage
    └── containerd
```

The directory has been created succesfully in `$HOME` (users home directory).

As you can see via the following output, the binaries for `docker`, `kind` as well as `kubectl` aren't installed on my local system:

```
which docker
docker not found

which kind
kind not found

which kubectl
kubectl not found
```

Asking how `vctl` `kind` will `--help` us here.

```
vctl kind --help

Get system environment ready for vctl-based KIND.
Using vctl as the provider for KIND instead of Docker, set up system environment to be ready for vctl-based KIND.
* KIND will be downloaded and installed if it's not detected.
* All Docker commands will be aliased to vctl in the current terminal.
* System is only configured for current terminal. All configuration for KIND will be lost when the terminal is closed.

USAGE:
  vctl kind [OPTIONS]

OPTIONS:
  -h, --help   Help for kind
```

There is some really important information in here!
1. If `kind` wasn't already installed before, `vctl` will download and install it for you.
2. In the current terminal session only, all `docker` commands will be aliased to `vctl`.
3. All made configurations apply only to the running terminal session.

```
vctl kind
Downloading 3 files...
Downloading [crx.vmdk 99.20% kubectl 62.50%]
Finished crx.vmdk 100.00%
Downloading [kubectl 100.00%]
Finished kubectl 100.00%
Downloading [kind-darwin-amd64 92.55%]
Finished kind-darwin-amd64 100.00%
3 files successfully downloaded.
vctl-based KIND is ready now. KIND will run local Kubernetes clusters by using vctl containers as "nodes"
* All Docker commands has been aliased to vctl in the current terminal. Docker commands performed in current window would be executed through vctl. If you need to use regular Docker commands, please use a separate terminal window.
sessions should be nested with care, unset $TMUX to force
```

As you can see on the previous terminal output, the command will download the necessary binaries for `kind` and `kubectl` as well as a `crx.vmdk` which will be used by the CRX VM.

{{< admonition type=quote title="VMware Documentation" open=true >}}
So in the Terminal window, the three binaries under user home `~/.vctl/bin` folder will take precedence over other existing versions of `kubectl/kind/docker` binaries that were installed before.
{{< /admonition >}}

Aha! This means that in case I already had e.g. `kubectl` installed on my system, the execution of `vctl kind` will affect my current running terminal by adjusting the `$PATH` for the according binaries. Let me validate this by leveraging `which`(*) again:

**The which utility takes a list of command names and searches the `$PATH` for each executable file.*

```
which kubectl
/Users/rguske/.vctl/bin/kubectl

which docker
/Users/rguske/.vctl/bin/docker

which kind
/Users/rguske/.vctl/bin/kind
```

That looks good to me! Also the `~/.vctl` directory has been filled up too.

```
tree -L 2
.
├── Fusion\ Container\ Storage.sparseimage
├── bin
│   ├── crx.vmdk
│   ├── docker -> /Applications/VMware\ Fusion.app/Contents/Library/vkd/bin/vctl
│   ├── kind
│   └── kubectl
├── config.json
├── config.toml
├── config.yaml
├── containerd.log
├── opt
│   └── containerd
└── storage
    └── containerd

5 directories, 9 files
```

### KinD create Cluster
It's time now to create a Kubernetes node. **Note!** `vctl system start` has to be executed first before leveraging `kind` for a deployment! Otherwise you will get the following error message:

{{< admonition type=failure title="Error" open=true >}}
Failed to list clusters: command "docker ps -a --filter label=io.x-k8s.kind.cluster=kind-1.19.1 --format '{{.Names}}'" failed with error: exit status 1
Command Output: level=fatal msg="Container engine is not running. Run 'vctl system start' to launch it.
{{< /admonition >}}

`vctl` assigns 2 GB of memory and 2 CPU cores by default for the CRX VM that hosts the node container. You can adjust the default configuration with the options `--k8s-cpus` and `--k8s-mem`. To create the Kubernetes node, basically `kind create cluster` is all you need to get started. If you like to give the node(s) a name or if you like to use a specific Kubernetes version which should run on your node, checkout the official [Docker repository](https://hub.docker.com/r/kindest/node/tags), pick your version and run e.g. `kind create cluster --image kindest/node:v1.19.1 --name kind-1.19.1`.

After a minute or two, you will have a Kubernetes node running on your desktop locally.

```
kubectl get all -A
NAMESPACE            NAME                                                    READY   STATUS    RESTARTS   AGE
kube-system          pod/coredns-f9fd979d6-spv95                             1/1     Running   0          116m
kube-system          pod/coredns-f9fd979d6-vh22z                             1/1     Running   0          116m
kube-system          pod/etcd-kind-1.19.1-control-plane                      1/1     Running   0          116m
kube-system          pod/kindnet-2jjmc                                       1/1     Running   0          116m
kube-system          pod/kube-apiserver-kind-1.19.1-control-plane            1/1     Running   0          116m
kube-system          pod/kube-controller-manager-kind-1.19.1-control-plane   1/1     Running   0          116m
kube-system          pod/kube-proxy-6dnwg                                    1/1     Running   0          116m
kube-system          pod/kube-scheduler-kind-1.19.1-control-plane            1/1     Running   0          116m
local-path-storage   pod/local-path-provisioner-78776bfc44-pnfbl             1/1     Running   0          116m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  116m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   116m

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      1         1         1       1            1           <none>                   116m
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   116m

NAMESPACE            NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          deployment.apps/coredns                  2/2     2            2           116m
local-path-storage   deployment.apps/local-path-provisioner   1/1     1            1           116m

NAMESPACE            NAME                                                DESIRED   CURRENT   READY   AGE
kube-system          replicaset.apps/coredns-f9fd979d6                   2         2         2       116m
local-path-storage   replicaset.apps/local-path-provisioner-78776bfc44   1         1         1       116m
```

Again! When closing the terminal, the following applies:

{{< admonition type=info title="Info" open=true >}}
The vctl-based KIND context will be lost if you close the Terminal window.
Next time you want to interact with the Kubernetes clusters, run the vctl kind command.
{{< /admonition >}}

### Recording
The details of the last two sections can be watched via the following recording. I've used [asciinema](https://asciinema.org/) to record my terminal in and -output.

<script id="asciicast-kVHSXxOmq1p3xOtY5ob0uIo9a" src="https://asciinema.org/a/kVHSXxOmq1p3xOtY5ob0uIo9a.js" async></script>

## Conclusion
Before closing this post with my conclusion, I wanted to mention the following as well. Besides the addition of `kind` to the `vctl` utility, it also got some new options which e.g. allows you to `login` into a container registry like [Harbor](https://goharbor.io) for example as well as to manage `volume`s (currently only support `volume prune`).

```
vctl

USAGE:
  vctl COMMAND [OPTIONS]

COMMANDS:
  inspect     Return low-level information on objects.
  kind        Get system environment ready for vctl-based KIND.
  login       Log in to a registry.
  logout      Log out from a registry.
  volume      Manage volumes.
```

This is also documented in the [Documentation](https://docs.vmware.com/en/VMware-Fusion/12/com.vmware.fusion.using.doc/GUID-7A2B7599-AF68-4209-8ABA-99619D9B8777.html).

I enjoyed digging deeper into the implementation details of Project Nautilus which is now fully integrated in Fusion 12 and Workstation 16. This is a great addition to VMware's Desktop Hypervisor solutions and provides a great experience not even for those who want to get started with containers and Kubernetes but also for the more experienced users among us.

## Resources
- Propose an Idea: [LINK](https://communities.vmware.com/community/vmtn/fusion/content?filterID=contentstatus[published]~objecttype~objecttype[idea])
- Join the Slack channel [**#fusion-workstation**](https://vmwarecode.slack.com/)

### #Fusion
- [Release Notes](https://docs.vmware.com/en/VMware-Fusion/12/rn/VMware-Fusion-12-Release-Notes.html#What's%20New)
- [Download](https://my.vmware.com/web/vmware/downloads/info/slug/desktop_end_user_computing/vmware_fusion/12_0)
- [FAQ](https://www.vmware.com/products/fusion/faq.html)
- File a bug on Github: [LINK](https://github.com/VMwareFusion/vctl-docs/issues/new)

### #Workstation
- [Release Notes](https://docs.vmware.com/en/VMware-Workstation-Pro/16/rn/VMware-Workstation-16-Pro-Release-Notes.html)
- [Download](https://my.vmware.com/de/web/vmware/downloads/info/slug/desktop_end_user_computing/vmware_workstation_player/16_0)
- [FAQ](https://www.vmware.com/products/workstation-pro/faq.html)
