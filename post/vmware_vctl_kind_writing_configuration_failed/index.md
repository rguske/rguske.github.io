# VMware vctl KinD - Writing Configuration Failed


## Introduction

{{< admonition quote "Quote" true >}}
**“Everything fails all the time”**

-- Werner Vogels, Amazon CTO
{{< /admonition >}}

^^But why always at the most inappropriate moments?! Just recently it happened to me, that one day before I needed my homelab for a presentation, the vSAN caching disk failed and had to be replaced.

<center> {{< tweet 1364258078410080270 >}} </center>

Normally, there isn't a real issue with loosing a caching device in a vSAN cluster but let me put it that way, I'm really using all my given 2-node cluster resources and a potential risk of data-loss was calculated. Not everybody can have a "homelab" like the Homelab King[^1], right? :wink:

Unfortunaltey, some of my Tanzu Kubernetes Grid clusters[^2] suffered from the outage and one of those clusters was the one, which is providing my Harbor[^3] registry service which I needed for my demo.

However, there are other ways to help yourself when your homelab is in "maintenance" and I wanted to leverage VMware Fusion `vctl`[^4] to quickly instantiate a KinD[^5] cluster locally on my machine, to install Harbor via a Helm chart[^6] afterwards on it. For the purposes of my presentation this would have been totally sufficient. But then it turned out that I have to find an alternative again :facepalm:.

## Adjusting the CRX VM with appropriate resources

By default `vctl` assigns 2 GB of memory and 2 CPU cores for the CRX VM that hosts the Kubernetes node container. For my Harbor one node deplyoment I've kept to the minimum requirements[^7] (2 CPUs, 4 GB Mem) for the Kubernetes node but also decided to give the CRX VM a little bit more, to keep the option open to run a Multo-Node deployment as well. Therefore, I  applied the following configuration:

`vctl system config --vm-cpus 4 --vm-mem 8192 --k8s-cpus 2 --k8s-mem 4096` which will adjust the `config.yaml` in `~/.vctl/` accordingly.

```yaml
cache-location: /Users/rguske/.vctl
k8s-cpus: 2
k8s-mem: 4096
log-level: info
log-location: /Users/rguske/.vctl/containerd.log
mount-name: Fusion Container Storage
pid: 45414
storage: 128g
vm-cpus: 4
vm-mem: 8192
vmnet: vmnet8
```

## KinD - Writing configuration failed

Moving forward to the KinD cluster creation itself. I first looked up for the `latest` [kindest/node](https://hub.docker.com/r/kindest/node/tags?page=1&ordering=last_updated) image version I can choose.

{{< image src="/img/posts/202102_vctlkindissue/CapturFiles-20210227_035317.jpg" caption="Figure I: kindest/node available images" src-s="/img/posts/202102_vctlkindissue/CapturFiles-20210227_035317.jpg" >}}

You can see at *Figure I* that **v1.19.7** is at the top (sort by: *Newest*) and so my choice was taken.

 `kind create cluster --image kindest/node:v1.19.7 --name harbor`

```code
Creating cluster "harbor" ...
 ✓ Ensuring node image (kindest/node:v1.19.7) 🖼
 ✓ Preparing nodes 📦
 ✗ Writing configuration 📜
ERROR: failed to create cluster: failed to generate kubeadm config content: failed to get kubernetes version from node: failed to get file: command "docker exec --privileged harbor-control-plane cat /kind/version" failed with error: exit status 1
Command Output: level=error msg="Error starting process: container = harbor-control-plane, command = [cat /kind/version] env = [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin TERM=xterm PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin container=docker TERM=xterm] error = container_linux.go:345: starting container process caused \"read init-p: connection reset by peer\": unknown"
```

What is this `error msg` telling us? Let's activate `debug` mode by adding option `-v 1` to the command.

`kind create cluster --image kindest/node:v1.19.7 --name harbor -v 1`

```
Creating cluster "harbor" ...
DEBUG: docker/images.go:58] Image: kindest/node:v1.19.7 present locally
 ✓ Ensuring node image (kindest/node:v1.19.7) 🖼
 ✓ Preparing nodes 📦
 ✗ Writing configuration 📜
ERROR: failed to create cluster: failed to generate kubeadm config content: failed to get kubernetes version from node: failed to get file: command "docker exec --privileged harbor-control-plane cat /kind/version" failed with error: exit status 1
Command Output: level=error msg="Error starting process: container = harbor-control-plane, command = [cat /kind/version] env = [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin TERM=xterm PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin container=docker TERM=xterm] error = container_linux.go:345: starting container process caused \"read init-p: connection reset by peer\": unknown"
Stack Trace:
sigs.k8s.io/kind/pkg/errors.WithStack
        sigs.k8s.io/kind/pkg/errors/errors.go:51
sigs.k8s.io/kind/pkg/exec.(*LocalCmd).Run
        sigs.k8s.io/kind/pkg/exec/local.go:124
sigs.k8s.io/kind/pkg/cluster/internal/providers/docker.(*nodeCmd).Run
        sigs.k8s.io/kind/pkg/cluster/internal/providers/docker/node.go:146
sigs.k8s.io/kind/pkg/exec.CombinedOutputLines
        sigs.k8s.io/kind/pkg/exec/helpers.go:67
sigs.k8s.io/kind/pkg/cluster/nodeutils.KubeVersion
        sigs.k8s.io/kind/pkg/cluster/nodeutils/util.go:35
sigs.k8s.io/kind/pkg/cluster/internal/create/actions/config.getKubeadmConfig
        sigs.k8s.io/kind/pkg/cluster/internal/create/actions/config/config.go:170
sigs.k8s.io/kind/pkg/cluster/internal/create/actions/config.(*Action).Execute.func1.1
        sigs.k8s.io/kind/pkg/cluster/internal/create/actions/config/config.go:82
sigs.k8s.io/kind/pkg/errors.UntilErrorConcurrent.func1
        sigs.k8s.io/kind/pkg/errors/concurrent.go:30
runtime.goexit
        runtime/asm_amd64.s:1374
```

I have to admit that I couldn't really use the output for further debugging and also searching on the web helped me neither. Consequently, I reached out internally and got the hint that **v1.20.0** should work.

**v1.20.0** :question: :exclamation: I haven't seen this version on the list of available images (*Figure I*) and so I went back to Docker Hub and searched for this particular version.

Okay, it exists and besides it a **v1.20.2**. as well. I learned *Newest* isn't related to Tags, what IMO would make sense, but it's related to *last pushed*.

{{< image src="/img/posts/202102_vctlkindissue/Screenshot2021-03-01.jpg" caption="Figure II: hidden kindest image version 1.20.0" src-s="/img/posts/202102_vctlkindissue/Screenshot2021-03-01.jpg" >}}

Give it a another try and this time with v1.20.0.

`kind create cluster --image kindest/node:v1.20.0 --name harbor`

```
Creating cluster "harbor" ...
 ✓ Ensuring node image (kindest/node:v1.20.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-harbor"
You can now use your cluster with:

kubectl cluster-info --context kind-harbor

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

```
kubectl get nodes -o wide
NAME                   STATUS   ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                                     KERNEL-VERSION       CONTAINER-RUNTIME
harbor-control-plane   Ready    control-plane,master   48s   v1.20.0   192.168.43.2   <none>        Ubuntu Groovy Gorilla (development branch)   4.19.138-7.ph3-esx   containerd://1.4.0
```

**Created.** On the one hand that's a satisfying result but on the other hand of course not, because we want to have the flexibilty to create a KinD cluster with every available Kubernetes (`kindest/node`) version.

If prior 1.20.0 won't work, will above 1.20.0 work?

 `kind create cluster --image kindest/node:v1.20.2 --name harbor`

```shell
Creating cluster "harbor" ...
 ✓ Ensuring node image (kindest/node:v1.19.7) 🖼
 ✓ Preparing nodes 📦
 ✗ Writing configuration 📜
ERROR: failed to create cluster: failed to generate kubeadm config content: failed to get kubernetes version from node: failed to get file: command "docker exec --privileged harbor-control-plane cat /kind/version" failed with error: exit status 1
Command Output: level=error msg="Error starting process: container = harbor-control-plane, command = [cat /kind/version] env = [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin TERM=xterm PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin container=docker TERM=xterm] error = container_linux.go:345: starting container process caused \"read init-p: connection reset by peer\": unknown"
```

:flushed: **Failed**

I repeated it with a couple of other versions and created the following table with my observations:

### My Test Results

|   **kindest/node Version** |   **Result** |
| :---: | :---: |
| 1.20.2 | :x: |
| 1.20.0 | :white_check_mark: |
| 1.19.7 | :x: |
| 1.19.1 | :white_check_mark: |
| 1.18.8 | :white_check_mark: |

All my tests were made with the following versions:

|   **OS, App, CLI, Runtime** |   **Version** |
| :---: | :---: |
| macOS Big Sur  | 11.2.1 (20D75) |
| Fusion | 12.1.0 (17195230) |
| vctl | 1.1.1 |
| containerd github.com/containerd/containerd | v1.3.2-vmw |

## Heads-up

{{< admonition warning "No solution yet" true >}}
I don't have a solution for this issue yet but I will keep this post updated. For the time being, I recommend using a version which passed my test.
{{< /admonition >}}

## Bonus: Multi-Node Deployment

I have mentioned at the beginning, that I have assigned enough resources to the CRX VM to run a multi-node KinD deployment and I'd like to show you now how to instantiate this by using the `--config` option.

Just create a file in a directory of your choice and add the following description to it:

**Example:**

`vim ~/.kube/kind_worker`

```
# two node (one ctrl plane & one worker node) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

`kind create cluster --image kindest/node:v1.20.0 --name kind-cluster-harbor --config ~/.kube/kind_worker`

```
Creating cluster "kind-cluster-harbor" ...
 ✓ Ensuring node image (kindest/node:v1.20.0) 🖼
 ✗ Preparing nodes 📦 📦 
ERROR: failed to create cluster: docker run error: command "docker run --hostname kind-cluster-harbor-control-plane --name kind-cluster-harbor-control-plane --label io.x-k8s.kind.role=control-plane --privileged --security-opt seccomp=unconfined --security-opt apparmor=unconfined --tmpfs /tmp --tmpfs /run --volume /var --volume /lib/modules:/lib/modules:ro --detach --tty --label io.x-k8s.kind.cluster=kind-cluster-harbor --net kind --restart=on-failure:1 --publish=127.0.0.1:61721:6443/TCP kindest/node:v1.20.0" failed with error: exit status 1
Command Output: level=error msg="failed to create container: container name (kind-cluster-harbor-control-plane) length is greater than the maximum allowed length (30)"
```

{{< admonition failure "container name length exeeded" true >}}
Command Output: level=error msg="failed to create container: container name (kind-cluster-harbor-control-plane) length is greater than the maximum allowed length (30)"
{{< /admonition >}}

Ooookay :roll_eyes: the given cluster name is too long. Shortened:

`kind create cluster --image kindest/node:v1.20.0 --name kind-harbor --config ~/.kube/kind_worker`

```
Creating cluster "kind-harbor" ...
 ✓ Ensuring node image (kindest/node:v1.20.0) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind-harbor"
You can now use your cluster with:

kubectl cluster-info --context kind-kind-harbor

Thanks for using kind! 😊
```

`kubectl get nodes -o wide`

```
NAME                        STATUS   ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                                     KERNEL-VERSION       CONTAINER-RUNTIME
kind-harbor-control-plane   Ready    control-plane,master   67s   v1.20.0   192.168.43.6   <none>        Ubuntu Groovy Gorilla (development branch)   4.19.138-7.ph3-esx   containerd://1.4.0
kind-harbor-worker          Ready    <none>                 39s   v1.20.0   192.168.43.5   <none>        Ubuntu Groovy Gorilla (development branch)   4.19.138-7.ph3-esx   containerd://1.4.0
kind-harbor-worker2         Ready    <none>                 39s   v1.20.0   192.168.43.7   <none>        Ubuntu Groovy Gorilla (development branch)   4.19.138-7.ph3-esx   containerd://1.4.0
```

Wohoo, a multi-node Kubernetes cluster running locally on my desktop.

## Commands I Used

These are the commands I used in this post:

- `vctl version`
- `vctl system config --vm-cpus 4 --vm-mem 8192 --k8s-cpus 2 --k8s-mem 4096`
- `kind create cluster --image kindest/node:v1.20.0 --name kind-cluster`
- `kind create cluster --image kindest/node:v1.20.0 --name kind-cluster -v 1`
- `vim ~/.kube/kind_worker`
- `kind create cluster --image kindest/node:v1.20.0 --name kind-harbor --config ~/.kube/kind_worker`

[^1]: [Homelab King Marc Huppert - HomeLab Actual State](https://vcdx181.com/homelab-actual-state/)
[^2]: [Provisioning Tanzu Kubernetes Clusters](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-2597788E-2FA4-420E-B9BA-9423F8F7FD9F.html)
[^3]: [Harbor Cloud Native Registry](https://goharbor.io)
[^4]: [A closer look at VMware's Project Nautilus](https://rguske.github.io/post/a-closer-look-at-vmwares-project-nautilus/)
[^5]: [KinD - Docker in Kubernetes](https://kind.sigs.k8s.io/)
[^6]: [vSphere 7 with Kubernetes supercharged - Helm, Harbor and Tanzu Kubernetes Grid](https://rguske.github.io/post/vsphere-7-with-kubernetes-supercharged-helm-harbor-tkg/)
[^7]: [Harbor Installation Prerequisites](https://goharbor.io/docs/1.10/install-config/installation-prereqs/)

