# Elevate Your Cloud-Native Journey: Knative the VMware Tanzu Way Part I - Streamlined Installation of Knative


## Knative :blue_heart:

In the rapidly evolving landscape of modern software architecture, event-driven systems have emerged as a pivotal paradigm, empowering organizations to create highly responsive, scalable, and adaptable applications. Knative stands at the forefront of this revolution, offering a robust and flexible framework for building event-driven architectures (EDA) that seamlessly integrate diverse components, enhance automation, and enable real-time data processing.

I'm evangelizing and supporting the [Knative project](https://knative.dev/docs/) for a longer time already. I recently had the pleasure to demonstarte parts of its comprehensive feature set at the great [ContainerDays](https://www.containerdays.io/) event in beautiful Hamburg, Germany.

<center> {{< tweet user="davidshadoow" id="1701219483145093496" >}} </center>

The overall positive feedback afterwards was gratifying and confirmed best that Knative is just awesome :rocket:.

Like it is often with exciting technologies which captivate me, the more I get into it, the more my excitement increases. But still, it feels like I'm just scratching the surface giving the fact, that the project continues to evolve and that it admittedly also has its complexity.

In my blog post [Demoing Knative Serving and Eventing using Demo-Magic-Scripts](https://rguske.github.io/post/demoing-knative-serving-and-eventing-using-demo-magic-scripts/), I barely touched two of the three building blocks of Knative, which are [Serving](https://knative.dev/docs/serving/), [Eventing](https://knative.dev/docs/eventing/) and [Functions](https://knative.dev/docs/functions/). But the intention of the post was more to promote the [demo-magic scripts](https://github.com/paxtonhare/demo-magic/tree/master) I've created in order to "easily" demo some Serving and Eventing goodness, as well as a recording of my VMware {Code} session.

Therefore, it's about time to start a new blog series and to share with you what I've touched on so far, which experiences I've made on my journey and most important, to provide resources, hints and tips & tricks which hopefully helps you on your journey with this thrilling project.

Naturally, such a series (IMO) has to start with guidance on how-to install Knative on an existing Kubernetes environment. Consequently, this first post will cover the installation of Knative.

The easiest way to get started is clearly [Knative Quickstart](https://knative.dev/docs/install/quickstart-install/), which is a plugin for the Knative cli `kn`. It'll install the minimium of the Serving and Eventing components on a local [KinD](https://github.com/kubernetes-sigs/kind) cluster. But of course this means, it's just for experimentation use only. Which is fine for the start!

Giving the fact that I'm working for VMware, I'm going to describe how to ***install Knative the VMware Tanzu way***.

## VMware Tanzu Cloud Native Runtimes

[Tanzu Cloud Native Runtimes](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/cnr-overview.html), commonly known as CNR, is VMware's supported Knative offering for Serving and Eventing. In a nutshell, it's a packaged, simple to deploy and with some VMware goodies equipped Knative version.

Simple to deploy because Day-1 as well as Day-2 operations of each building block (Serving, Eventing) will be done using the [Carvel](https://carvel.dev) tools suite. A comprehensive set of reliable, single-purpose tools which simplify the life-cycle of applications (*Figure I*).

{{< image src="/img/posts/202309_knative_part_1/202309_knative_part_1_carvel.png" caption="Figure I: Carvel Tools Overview" src-s="/img/posts/202309_knative_part_1/202309_knative_part_1_carvel.png" >}}

The mentioned "goodies" are covered in one of the next posts.

In order to manage internal and external access to the services in a cluster, CNR uses [Contour](https://projectcontour.io/), an open source Kubernetes ingress controller maintained by VMware.

As of writing this series, [CNR version 2.3.1](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/release-notes.html) is `latest` and also the version which I'm using for the sake of this post. It's recommended to check the CNR release notes before start installing CNR (as usual). CNR version 2.3.1 includes Knative 1.10.2 and supports Kubernetes version 1.25 and higher.

## Prerequisites

Here's the thing! Cloud Native Runtimes is not a single purchasable VMware solution. It's part of VMware's modular application development platform solution [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html) (TAP). It is used as the runtime foundation for TAP and provides many of the great Knative Serving features, like:

- scale from 0 to many and back to 0 (concurrency-based scaling)
- traffic routing/splitting
- traffic encryption

Therefore, the TAP documentation is the source of truth for topics like prerequisites and installation details. Please make sure to read it carefully and to meet the documented prerequisites.

- [Prerequisites for installing Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)

**TL;DR:**

- A [Tanzu Network](https://network.tanzu.vmware.com/) account to download Tanzu Application Platform packages
- Network access to https://registry.tanzu.vmware.com
- A container image registry, such as [Harbor](https://goharbor.io/) for application images, base images, and runtime dependencies
  - with enough storage capacity
- A wildcard subdomain for applications
- An appropriate Kubernetes cluster-size - [Resource requirements](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)
- Installation of the [Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.6/cluster-essentials/deploy.html) for non-Tanzu Kubernetes clusters (TKC)

**Important:**

If you are planning to install CNR on a non-Tanzu Kubernetes Grid (TKG = VMware's Kubernetes solution) cluster, please make sure to install the [Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.6/cluster-essentials/deploy.html) as described.

## :down_arrow::up_arrow: Relocation of Image-Bundles/Packages

One preperation for the actual installation requires the relocation of the necessary installation [image-bundles](https://carvel.dev/imgpkg/). Relocation describes the download and upload of an image-bundle. A bundle captures your configuration files and a list of references to images on which they depend.

Basically, it's not a "hard" requirement but it's recommended to relocate the bundles to your own private Cloud Native Registry, like Harbor, in order to be independed from the availability of the VMware Registry.

Two bundles can be relocated.

> Image Bundle(s) == Package(s)

- [Packages](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2/about-tkg/packages-index.html) for TKG
  - Packages includes certain Kubernetes cluster extensibilities like Antrea (CNI), Contour (Ingress), Harbor (Registry), cert-manager (cert issuer) and more
- TAP specific packages
  - necessary for the installation of TAP

Two options exist to achieve our goal. The first option is to download the packages as `tar` ball (`--to-tar`) and to ultimately upload the file.

{{< mermaid >}}
graph LR;
    A( User ) -->| imgpkg copy -b VMware Repo-URL --to-tar bundle.tar | B( Locally )
    B --> A
    A -->| imgpkg copy --tar bundle.tar --to-repo Harbor Repo-URL | C( Registry )
{{< /mermaid >}}

The second option is to directly copy the packages over from one registry to another (`--to-repo`).

{{< mermaid >}}
graph LR;
    A( User ) -->| imgpkg copy -b VMware Repo-URL --to-repo Harbor Repo-URL | B( Registry )
{{< /mermaid >}}

I dedicated a separate article on [Sharing Container Images using the Docker CLI or Carvel's imgpkg](https://rguske.github.io/post/sharing-container-images-using-the-docker-cli-or-carvels-imgpkg/). You'll find many helpful details in this post.

### Relocating Tanzu Kubernetes Grid Packages

The TKG-Packages can be relocated using the `tanzu` cli. Make sure you have at least 10G of disk space available before initiating the download process. Here's a working example:

**Download**:

Make sure you've created a folder beforehand, otherwise you'll float your `dir`.

```shell
tanzu isolated-cluster download-bundle \
--source-repo projects.registry.vmware.com/tkg \
--tkg-version v2.1.1
```

**Upload**:

The following command will upload the downloaded image-bundles (packages) to a referenced registry repository.

```shell
tanzu isolated-cluster upload-bundle \
--source-directory /home/vmware/packages \
--destination-repo harbor01.cpod-nsxv8.az-stc.cloud-garage.net/packages \
--ca-certificate /home/vmware/harbor01/ca.crt
```

I skipped the relocation of the (standard) Packages for TKG and kept the pre-configured online package repository for TKG.

### Relocating Tanzu Application Platform Packages - Option #1

Let's start relocating the packages for TAP.

1. Export specific metadata as variables:

```shell
export TAP_VERSION='1.6.3' \
export IMGPKG_REGISTRY_HOSTNAME='harbor01.cpod-nsxv8.az-stc.cloud-garage.net' \
export REGISTRY_CA_PATH='/home/vmware/stuff/harbor01/ca.crt' \
export IMGPKG_REGISTRY_USERNAME='admin' \
export IMGPKG_REGISTRY_PASSWORD='******'
```

2. Download the TAP packages:

```shell
imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$TAP_VERSION \
  --to-tar tap-packages-$TAP_VERSION.tar \
  --include-non-distributable-layers
```

If everything went well, the operation will end with the result `succeeded` and the tar-ball has a size of roughly 10G.

```shell
du -h tap-packages-1.6.3.tar

9,5G    tap-packages-1.6.3.tar
```

3. Upload the packages to your private registry instance:

```shell
imgpkg copy \
  --tar tap-packages-$TAP_VERSION.tar \
  --to-repo $IMGPKG_REGISTRY_HOSTNAME/tap/tap-packages-$TAP_VERSION \
  --registry-ca-cert-path $REGISTRY_CA_PATH \
  --include-non-distributable-layers
```

Hopefully a similar result applies to this operation as well.

```shell
copy | importing 265 images...

9.42 GiB / 9.43 GiB [--------------------------------------------------->] 99.91% 16.02 MiB p/s
copy |
copy | done uploading images
copy | Warning: '--include-non-distributable-layers' flag provided, but no images contained a non-distributable layer.
copy | Tagging images

Succeeded
```

Ultimately, all image bundles are showing up in our registry (*Figure II*).

{{< image src="/img/posts/202309_knative_part_1/202309_knative_part_1_packages.png" caption="Figure II: Uploaded TAP Packages in Harbor" src-s="/img/posts/202309_knative_part_1/202309_knative_part_1_packages.png" >}}

### Relocating Tanzu Application Platform Packages - Option #2

Like initially introduced, the 2nd option is to directly `copy` the images over. The first option helps in case you are running an internet-restricted environment (air-gapped) and the image-bundles have to be shared via organisation comliant ways.

```shell
imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$TAP_VERSION \
  --to-repo harbor01.cpod-nsxv8.az-stc.cloud-garage.net/tap/tap-packages-$TAP_VERSION \
  --registry-ca-cert-path=$REGISTRY_CA_PATH
```

The result is identical to option #1. With this, three check marks can be set on the prerequisites:

- Access to Tanzu Network :white_check_mark:
- Network access to VMware registry :white_check_mark:
- Relocation of the Tanzu Application Packages :white_check_mark:

Next - Tazu Package installation preperations.

## Preperations for Tanzu Packages Installation

As aforementioned, I'm using VMware Tanzu Kubernetes Grid to instantiate Kubernetes clusters on vSphere and I've also mentioned that I'm going to use the Carvel tools to install certain Kubernetes cluster extensions (Ingress, etc.). The [kapp-controller](https://carvel.dev/kapp-controller/) is a crucial component which needs to run in your k8s cluster. It runs by default on every deployed Tanzu Kubernetes Cluster (TKC).

```shell
k -n tkg-system get deploy
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
kapp-controller                         1/1     1            1           108m
tanzu-capabilities-controller-manager   1/1     1            1           106m
```

As listed in the [prerequisites](#prerequisites) section, install the Cluster Essentials on Kubernetes clusters which aren't TKG, to have e.g. the kapp-controller available as well.

Furthermore, in order to easily distribute secrets across namespaces, the [secretgen-controller](https://github.com/carvel-dev/secretgen-controller) is used and also installed on TKCs by default.

```shell
k -n secretgen-controller get deploy
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
secretgen-controller   1/1     1            1           107m
```

### Adding the TAP Package Repository

I won't cover Tanzu Packages specificly in this post. In case you aren't familiar with it, read up on it [HERE](https://beyondelastic.com/2022/01/04/tanzu-packages-explained/), [HERE](https://rguske.github.io/post/deploying-tanzu-packages-using-tanzu-mission-control-catalog/) and [HERE](https://rguske.github.io/post/deploy-tanzu-packages-from-a-private-registry/).

Per default, the custom resource (CR) `packagerepositories.packaging.carvel.dev` named `tanzu-standard` within the `tkg-system` namespace...

```shell
k get packagerepositories.packaging.carvel.dev -n tkg-system

NAME             AGE   DESCRIPTION
tanzu-standard   27d   Reconcile succeeded
```

... is configured with the url for the Tanzu Packages for TKG. See:

```shell
tanzu package repository list -n tkg-system

NAME            REPOSITORY                                                       TAG                  STATUS               DETAILS
tanzu-standard  extensions.aws-usw2.tmc.cloud.vmware.com/packages/standard/repo  v2023.7.13_update.2  Reconcile succeeded
```

By using the command `tanzu package available list -n tkg-system`, the kapp-controller will retrieve all available packages from the configured url. We want to have the same for the relocated TAP packages.

We start with the creation of a new namespace for the TAP packages.

```shell
k create ns tap-packages

namespace/tap-packages created
```

This namespace will be used to store TAP packages related resources (`configmaps`, `secrets`, etc.) within the namespace.

```shell
tanzu package repository add tap-1.6.3 --url harbor01.cpod-nsxv8.az-stc.cloud-garage.net/tap/tap-packages-1.6.3 -n tap-packages
ℹ   Adding package repository 'tap-1.6.3'
ℹ   Validating provided settings for the package repository
ℹ   Creating package repository resource
ℹ   Waiting for 'PackageRepository' reconciliation for 'tap-1.6.3'
\ 'PackageRepository' resource install status: Reconciling!

Please consider using 'tanzu package repository update' to update the package repository with correct settings

ℹ   'PackageRepository' resource install status: Reconciling


Error: resource reconciliation failed: vendir: Error: Syncing directory '0':
  Syncing directory '.' with imgpkgBundle contents:
    Error while preparing a transport to talk with the registry:
      Unable to create round tripper:
        Get "https://harbor01.cpod-nsxv8.az-stc.cloud-garage.net/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
. Reconcile failed: Fetching resources: Error (see .status.usefulErrorMessage for details)
```

If your kapp-controller is not configured to trust your private registry, the operations will quickly end with the shown error.

{{< admonition bug "x509 Certificate Error" true >}}
Get "https://harbor01.cpod-nsxv8.az-stc.cloud-garage.net/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
{{< /admonition >}}

I've covered this topic in my blog post [Deploy VMware Tanzu Packages from a private Container Registry](https://rguske.github.io/post/deploy-tanzu-packages-from-a-private-registry/#optional-kapp-controller-secret-vs-configmap).

In order to have the kapp-controller trust my registry, I have to present the registry certificate via a secret.

```yaml
kubectl -n tkg-system create -f - <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: kapp-controller-config
  namespace: tkg-system
stringData:
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIDKDCCAhCgAwIBAgIQJBt7sHm36rcMe4G8l3WytjANBgkqhkiG9w0BAQsFADAU
    MRIwEAYDVQQDEwlIYXJib3IgQ0EwHhcNMjMwNDI2MDc0NTUyWhcNMzMwNDIzMDc0
  [...]
  httpProxy: ""
  httpsProxy: ""
  noProxy: ""
  dangerousSkipTLSVerify: ""
EOF
```

```shell
k -n tkg-system get secret
NAME                     TYPE                             DATA   AGE
kapp-controller-config   Opaque                           5      8s
tanzu-standard-fetch-0   kubernetes.io/dockerconfigjson   1      27d
```

Delete the kapp-controller pod to pick up the additional configuration.

```shell
k -n tkg-system delete po kapp-controller-57d6c9d7bf-b749v

pod "kapp-controller-57d6c9d7bf-b749v" deleted
```

Due to the failed adding of the TAP packagerepository, the CR is still in `Reconcile failed` state. Let's validate if the updated kapp-controller configuration works by using the command `tanzu package repository update`.

```shell
tanzu package repository update tap-1.6.3 --url harbor01.cpod-nsxv8.az-stc.cloud-garage.net/tap/tap-packages-1.6.3 -n tap-packages
ℹ   Updating package repository 'tap-1.6.3'
ℹ   Getting package repository 'tap-1.6.3'
ℹ   Validating provided settings for the package repository
ℹ   Updating package repository resource
ℹ   Waiting for 'PackageRepository' reconciliation for 'tap-1.6.3'
ℹ   'PackageRepository' resource install status: ReconcileSucceeded
ℹ   'PackageRepository' resource successfully reconciled
ℹ  Updated package repository 'tap-1.6.3' in namespace 'tap-packages'
```

Looks better already. Checking the ultimate state:

```shell
tanzu package repository list -n tap-packages

  NAME       REPOSITORY                                                          TAG       STATUS               DETAILS
  tap-1.6.3  harbor01.cpod-nsxv8.az-stc.cloud-garage.net/tap/tap-packages-1.6.3  (>0.0.0)  Reconcile succeeded
```

Perfect! Now, lets check which packages are available using `tanzu package available list -n tap-packages`.

**Hint:** Make sure to run a decent screen resolution :wink:

{{< image src="/img/posts/202309_knative_part_1/202309_knative_part_1_tap_packages.png" caption="Figure III: All available Tanzu Application Platform Packages" src-s="/img/posts/202309_knative_part_1/202309_knative_part_1_tap_packages.png" >}}

That's quite A LOT of packages but keep calm, not all are necessarely needed for TAP. Positively speaking, it is good to only have one repository to maintain, one source of truth for TAP.

Furthermore, if it is TAP related, like Cloud Native Runtimes, make sure to only use packages provided by the TAP packages repository.

Example: Here's a direct comparison of the provided cert-manager versions for the two available package repositories `tanzu-standard` and `tap-1.6.3`.

| Tanzu Standard - Cert-Manager Version | TAP Packages - Cert-Manager Version |
| :-: | :-: |
| 1.11.1+vmware.1-tkg.1 | 2.3.1 |

You can notice that the available versions for cert-manager differs from each other.

**Preperations:**

- Adding the TAP Package Repository :white_check_mark:

## Installation of Tanzu Packages

This section describes the actual installation of **Cloud Native Runtimes** with having the above mentioned prerequisites and preperations in place. Packages which I'm going to install are:

- cert-manager.tanzu.vmware.com
- contour.tanzu.vmware.com
- cloud-native-runtimes.tanzu.vmware.com
- eventing.tanzu.vmware.com

Normally, the first three listed packages are installed when following the TAP installation routine. This is not my intention and therefore I have to install the packages one by one. Also, this will affect the `values.yaml` file for the CNR installation itself, since we have to specify that we are using an existing Contour installation.

Two different ways can be used to operate packages. You can either use the `tanzu` cli or the `kubectl` cli. For the sake of showing how this looks like, I'm going to demonstrate it for the packages cert-manager and Contour.

### Installing Cert-Manager

The certificate-controller cert-manager is the first component which will be installed.

- Using the `tanzu` cli:

```shell
tanzu package install cert-manager --package-name cert-manager.tanzu.vmware.com --version 2.3.1 -n tap-packages --wait

ℹ   Installing package 'cert-manager.tanzu.vmware.com'
ℹ   Getting package metadata for 'cert-manager.tanzu.vmware.com'
ℹ   Creating service account 'cert-manager-tap-packages-sa'
ℹ   Creating cluster admin role 'cert-manager-tap-packages-cluster-role'
ℹ   Creating cluster role binding 'cert-manager-tap-packages-cluster-rolebinding'
ℹ   Creating package resource
ℹ   Waiting for 'PackageInstall' reconciliation for 'cert-manager'
ℹ   'PackageInstall' resource install status: Reconciling
ℹ   'PackageInstall' resource install status: ReconcileSucceeded
ℹ
 Added installed package 'cert-manager'
```

- Using `kubectl`:

```yaml
kubectl create -f - <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-manager-sa
  namespace: tap-packages
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: cert-manager-sa
    namespace: tap-packages
---
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageInstall
metadata:
  name: cert-manager
  namespace: tap-packages
spec:
  serviceAccountName: cert-manager-sa
  packageRef:
    refName: cert-manager.tanzu.vmware.com
    versionSelection:
      constraints: 2.3.1
  values:
  - secretRef:
      name: cert-manager-data-values
---
apiVersion: v1
kind: Secret
metadata:
  name: cert-manager-data-values
  namespace: tap-packages
stringData:
  values.yml: |
    ---
    namespace: cert-manager
EOF
```

The installation can be validated checking the `packageinstall` CR.

```shell
tanzu package installed list -n tap-packages

NAME                   PACKAGE-NAME                   PACKAGE-VERSION        STATUS
cert-manager           cert-manager.tanzu.vmware.com  2.3.1                  Reconcile succeeded
```

```shell
k -n tap-packages get packageinstall

NAME           PACKAGE NAME                    PACKAGE VERSION         DESCRIPTION           AGE
cert-manager   cert-manager.tanzu.vmware.com   2.3.1                   Reconcile succeeded   34m
```

### Installing Contour

Contour will be deployed the same way like cert-manager but since Contour provides a few more [configurations for the installation](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-packages-ingress-contour.html#configs), a dedicated `values.yaml` file will be used. I created a new file named `contour-data-values.yaml` and added the following content:

```yaml
---
infrastructure_provider: vsphere
namespace: tanzu-system-ingress
contour:
  configFileContents: {}
  useProxyProtocol: false
  replicas: 2
  pspNames: "vmware-system-restricted"
  logLevel: info
envoy:
  service:
    type: LoadBalancer
    annotations: {}
    externalTrafficPolicy: Cluster
    disableWait: false
  hostPorts:
    enable: false
    http: 80
    https: 443
  hostNetwork: false
  terminationGracePeriodSeconds: 300
  logLevel: info
  pspNames: ""
certificates:
  duration: 8760h
  renewBefore: 360h
```

By default the `service type` for Envoy is `NodePort` but since I'm installing Contour to a vSphere cluster that uses NSX (same applies to the NSX Advanced Load Balancer) as a load balancer service provider, I adjusted this option to use `LoadBalancer` instead.

The installation will be kicked-off by executing:

```shell
tanzu package install contour --package-name contour.tanzu.vmware.com --version 1.24.4+vmware.1-tkg.1 -n tap-packages --values-file contour-data-values.yaml --wait

ℹ   Installing package 'contour.tanzu.vmware.com'
ℹ   Getting package metadata for 'contour.tanzu.vmware.com'
ℹ   Creating service account 'contour-tap-packages-sa'
ℹ   Creating cluster admin role 'contour-tap-packages-cluster-role'
ℹ   Creating cluster role binding 'contour-tap-packages-cluster-rolebinding'
ℹ   Creating secret 'contour-tap-packages-values'
ℹ   Creating package resource
ℹ   Waiting for 'PackageInstall' reconciliation for 'contour'
ℹ   'PackageInstall' resource install status: Reconciling
ℹ   'PackageInstall' resource install status: ReconcileSucceeded
ℹ
 Added installed package 'contour'
```

- The `kubectl` way:

```yaml
kubectl create -f - <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: contour-sa
  namespace: tap-packages
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: contour-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: contour-sa
    namespace: tap-packages
---
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageInstall
metadata:
  name: contour
  namespace: tap-packages
spec:
  serviceAccountName: contour-sa
  packageRef:
    refName: contour.tanzu.vmware.com
    versionSelection:
      constraints: 1.24.4+vmware.1-tkg.1
  values:
  - secretRef:
      name: contour-values
---
apiVersion: v1
kind: Secret
metadata:
  name: contour-values
  namespace: tap-packages
stringData:
  values.yaml: |
    contour:
     configFileContents: {}
     useProxyProtocol: false
     replicas: 2
     pspNames: vmware-system-restricted
     logLevel: info
    envoy:
     service:
       type: LoadBalancer
       annotations: {}
       externalTrafficPolicy: Cluster
       disableWait: false
     hostPorts:
       enable: false
       http: 80
       https: 443
     hostNetwork: false
     terminationGracePeriodSeconds: 300
     logLevel: info
    certificates:
     duration: 8760h
     renewBefore: 360h
EOF
```

Validating it at the end:

```shell
k -n tap-packages get packageinstall
NAME           PACKAGE NAME                    PACKAGE VERSION         DESCRIPTION           AGE
cert-manager   cert-manager.tanzu.vmware.com   2.3.1                   Reconcile succeeded   4h52m
contour        contour.tanzu.vmware.com        1.24.4+vmware.1-tkg.1   Reconcile succeeded   4h18m
```

## DNS Wildcard Subdomain

Knative allocates a wildcard subdomain for Knative Services. What is a Knative Service? I'm planning to create a dedicated blog post on Knative Serving in which this specific resource will be part of. For now, a Knative Service describes an application like a webserver or a function for example which got deployed using Knative.

In my environment, DNS services are provided using [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq#:~:text=dnsmasq%20is%20free%20software%20providing,dnsmasq). All future workloads which will run on my Knative installation will use this subdomain. Using a wildcard subdomain like e.g. `jarvis.cpod-nsxv8.az-stc.cloud-garage.net` requires an IP address which can be used for proper resolution. This IP address is provided by the high-performnce, robust and flexible proxy Envoy which comes with the Contour installation.

The IP address can be obtained via:

```shell
kubectl get service envoy -n tanzu-system-ingress --output 'jsonpath={.status.loadBalancer.ingress}'

[{"ip":"10.15.8.7"}]
```

I'm gpoing to use the given IP address to create the needed entry in my `/etc/dnsmasq.conf` file:

```shell
vim /etc/dnsmasq.conf

[...]
address=/jarvis.cpod-nsxv8.az-stc.cloud-garage.net/10.15.8.7
```

A restart of the `dnsmasq.service` is required to take effect on the made changes (`systemctl restart dnsmasq.service`).

## Securing Workloads in CNR

Multiple options are provided to secure `http` connections to your instantiated applications. Since cert-manager is already installed to add certificates and certificate issuers as resource types to Kubernetes clusters, we can use it to obtain, renew, and use our own certificates.

I created a new [shared ingress issuer](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/security-and-compliance-tls-and-certificates-ingress-issuer.html) as an on-platform representation of my certificate authority. Therefore, all new workloads get their ingress certificates issued by the *shared ingress issuer*.

The first step is to create a secret type `kubernetes.io/tls`. This secret contains "myCA" `tls.crt` as well as `tls.key` data.

```yaml
kubectl create -f - <<EOF
---
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: my-company-ca
  namespace: cert-manager
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIID9TCCAt2gAwIBAgIUQpwKWpTwO42xl6JpL1Xk1Vn6KjcwDQYJKoZIhvcNAQEL
    [...]
  tls.key: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEAr3NTAVllxJBzOw9+2guOnzt+kmZrbC88NFJpl8fkbC5WmgB8
    [...]
```

This step is followed by the creation of the `ClusterIssuer` itself. I simply named it `my-company`.

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-company
spec:
  ca:
    secretName: my-company-ca
```

This issuer has the earlier created `tls` certificate `my-company-ca` configured.

### Another way to use your own TLS Certificates

It is also possible to bypass the `ClusterIssuer` and use the Contour CR `TLSCertificateDelegation` instead. This flexibility makes it possible to provide custom certificates and delegate these to your `targetNamespaces`.

Here's a working example:

```shell
kubectl create -f - <<EOF
---
apiVersion: projectcontour.io/v1
kind: TLSCertificateDelegation
metadata:
  name: default-delegation
  namespace: cert-manager
spec:
  delegations:
  - secretName: my-company-ca
    targetNamespaces:
    - "my-app-namespace"
EOF
```

*Source:* [Use your existing TLS Certificate for Cloud Native Runtimes](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/knative-default-tls.html)

### Use wildcard certificates with Cloud Native Runtimes

And just another option which is available to explore is to set up a custom ingress issuer that issues wildcard certificates.

Look it up on VMware Docs: [Use wildcard certificates with Cloud Native Runtimes](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/tls-guides-wildcard-cert.html).

## Installing Cloud Native Runtimes :rocket:

Having everything settled and in place, I can start achieving the ultimate goal, the installation of Knative Serving via VMware's Cloud Native Runtimes. Similar to what I used for the Contour package, I also have to provide a `values.yaml` file to apply certain configurations for the installation.

My configuration includes the following:

| Key | Type | Description |
| :-: | :-: | :-: |
| `domain_name` | "jarvis.cpod-nsxv8.az-stc.cloud-garage.net" | Wildcard domain name for Knative Services which I configured with the assigned load balancer IP address of the Envoy proxy. |
| `domain_template` | "{{.Name}}.{{.Domain}}" | This option gives you the flexibility to construct the DNS name for the Knative Service. |
| `ingress_issuer` | "cpod-jarvis" | Using this option sets the Auto-TLS feature to `true`. |
| `ingress` (internal and external) | "tanzu-system-ingress" | Configures Contour to route ingress traffic for external and internal services. |

Create the file and add the following configurations (adjusted properly) to it.

```yaml
---
domain_name: "jarvis.cpod-nsxv8.az-stc.cloud-garage.net"
domain_template: "{{.Name}}.{{.Domain}}"
---
ingress_issuer: "cpod-jarvis"
---
ingress:
  external:
    namespace: "tanzu-system-ingress"
  internal:
    namespace: "tanzu-system-ingress"
```

And off we go:

```shell
tanzu package install cloud-native-runtimes \
-p cnrs.tanzu.vmware.com \
-v 2.3.1 \
-n tap-packages \
-f values.yaml \
--poll-timeout 30m
```

Oh, oh, no Kubernetes deployments or pods showing up...

### Pod Security Admission Controller :punch:

After a long breath of waiting for pods finally showing up on my cluster, I got troubled. The Pods Security Policy admission controller hitted me a couple of times already. Therefore, I checked one of the created `replicaSets` within the `Knative-Serving` namespace and there I found it:

```shell
[...]

message: 'pods "activator-65b6db76d7-8wmh4" is forbidden: violates PodSecurity
  "restricted:latest": seccompProfile (pod or container "activator" must set securityContext.seccompProfile.type
```

Since I'm running a Kubernetes v1.26 cluster in my environment...

```shell
k get nodes
NAME                                       STATUS   ROLES           AGE   VERSION
rguske-cnr-1-lgf5s-hftsg                   Ready    control-plane   30h   v1.26.5+vmware.2-fips.1
rguske-cnr-1-lgf5s-ldrsw                   Ready    control-plane   30h   v1.26.5+vmware.2-fips.1
rguske-cnr-1-lgf5s-q8jfl                   Ready    control-plane   30h   v1.26.5+vmware.2-fips.1
rguske-cnr-1-md-0-9z52m-5878bb7888-dtdnd   Ready    <none>          30h   v1.26.5+vmware.2-fips.1
rguske-cnr-1-md-0-9z52m-5878bb7888-xl769   Ready    <none>          30h   v1.26.5+vmware.2-fips.1
rguske-cnr-1-md-0-9z52m-5878bb7888-xm2xt   Ready    <none>          30h   v1.26.5+vmware.2-fips.1
```

...the Pods Security Policy admission controller isn't used anymore. The new Pod Security admission controller was introduced in Kubernetes v1.25 and is the "successor" of the Pods Security Policy admission controller. Consequently, I configured the pod-security with `privileged` for all my namespaces:

```shell
kubectl label --overwrite ns --all pod-security.kubernetes.io/enforce=privileged

namespace/cert-manager labeled
namespace/default labeled
namespace/knative-serving labeled
namespace/kube-node-lease labeled
namespace/kube-public labeled
namespace/kube-system labeled
namespace/secretgen-controller labeled
namespace/tanzu-system-ingress not labeled
namespace/tap-packages labeled
namespace/tkg-system labeled
namespace/vmware-system-antrea not labeled
namespace/vmware-system-auth not labeled
namespace/vmware-system-cloud-provider labeled
namespace/vmware-system-csi not labeled
namespace/vmware-system-tkg labeled
namespace/vmware-system-tmc not labeled
```

*Source:* [Configure Pod Security for TKR 1.25 and Later](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-B57DA879-89FD-4C34-8ADB-B21CB3AE67F6.html)

I had to wait a moment until the `reconciliation` operation kicked in again for the new installed package. You can always check the status of the `packageinstalls.packaging.carvel.dev` CR, short `app`.

```shell
k get app -A
NAMESPACE           NAME                                      DESCRIPTION           SINCE-DEPLOY   AGE
tap-packages        cert-manager                              Reconcile succeeded   8m58s          2d5h
tap-packages        cloud-native-runtimes                     Reconcile succeeded   8m49s          47h
tap-packages        contour                                   Reconcile succeeded   40s            2d5h
vmware-system-tkg   rguske-cnr-1-antrea                       Reconcile succeeded   6m37s          2d6h
vmware-system-tkg   rguske-cnr-1-capabilities                 Reconcile succeeded   4m18s          2d6h
vmware-system-tkg   rguske-cnr-1-guest-cluster-auth-service   Reconcile succeeded   7m57s          2d6h
vmware-system-tkg   rguske-cnr-1-metrics-server               Reconcile succeeded   7m28s          2d6h
vmware-system-tkg   rguske-cnr-1-pinniped                     Reconcile succeeded   8m             2d6h
vmware-system-tkg   rguske-cnr-1-secretgen-controller         Reconcile succeeded   4m15s          2d6h
vmware-system-tkg   rguske-cnr-1-vsphere-cpi                  Reconcile succeeded   5m14s          2d6h
vmware-system-tkg   rguske-cnr-1-vsphere-pv-csi               Reconcile succeeded   32s            2d6h
```

## Update Cloud Native Runtimes configuration

There are many package configuration options exposed through data values that allows you to customize your Cloud Native Runtimes installation. The available values can be obtained executing:

```shell
export CNR_VERSION=2.3.1
tanzu package available get cnrs.tanzu.vmware.com/${CNR_VERSION} --values-schema -n tap-packages > cnr-values-schema.yaml
```

Also, if you want to check which values were used for the initial installation of a package, you can get it from the corresponding `secret` inside the namespace which were used for the package. Obvisouly, the values aren't available in plain text. However, you can simply decode the `base64` encoded data.

Here's an example for the CNR package.

- First: Retrieve the encoded data

```shell
k -n tap-packages get secret cloud-native-runtimes-tap-packages-values -oyaml

apiVersion: v1
data:
  values.yaml: LS0tCmRvbWFpbl9uYW1lOiAiamFydmlzLmNwb2QtbnN4djguYXotc3RjLmNsb3VkLWdhcmFnZS5uZXQiCmRvbWFpbl90ZW1wbGF0ZTogInt7Lk5hbWV9fS57ey5Eb21haW59fSIKIy0tLQojZGVmYXVsdF90bHNfc2VjcmV0OiAiY2VydC1tYW5hZ2VyL215LWNvbXBhbnktY2EiCi0tLQppbmdyZXNzX2lzc3VlcjogImNwb2QtamFydmlzIgotLS0KaW5ncmVzczoKICBleHRlcm5hbDoKICAgIG5hbWVzcGFjZTogInRhbnp1LXN5c3RlbS1pbmdyZXNzIgogIGludGVybmFsOgogICAgbmFtZXNwYWNlOiAidGFuenUtc3lzdGVtLWluZ3Jlc3MiCg==
kind: Secret
metadata:
  creationTimestamp: "2023-10-02T19:56:13Z"
  name: cloud-native-runtimes-tap-packages-values
  namespace: tap-packages
  resourceVersion: "663595"
  uid: 0c2d9e9e-6c63-4495-8762-76bbb59eeabd
type: Opaque
```

- Second: Decode the data

```shell
`echo 'LS0tCmRvbWFpbl9uYW1lOiAiamFydmlzLmNwb2QtbnN4djguYXotc3RjLmNsb3VkLWdhcmFnZS5uZXQiCmRvbWFpbl90ZW1wbGF0ZTogInt7Lk5hbWV9fS57ey5Eb21haW59fSIKIy0tLQojZGVmYXVsdF90bHNfc2VjcmV0OiAiY2VydC1tYW5hZ2VyL215LWNvbXBhbnktY2EiCi0tLQppbmdyZXNzX2lzc3VlcjogImNwb2QtamFydmlzIgotLS0KaW5ncmVzczoKICBleHRlcm5hbDoKICAgIG5hbWVzcGFjZTogInRhbnp1LXN5c3RlbS1pbmdyZXNzIgogIGludGVybmFsOgogICAgbmFtZXNwYWNlOiAidGFuenUtc3lzdGVtLWluZ3Jlc3MiCg==' | base64 -d`

---
domain_name: "jarvis.cpod-nsxv8.az-stc.cloud-garage.net"
domain_template: "{{.Name}}.{{.Domain}}"
#---
#default_tls_secret: "cert-manager/my-company-ca"
---
ingress_issuer: "cpod-jarvis"
---
ingress:
  external:
    namespace: "tanzu-system-ingress"
  internal:
    namespace: "tanzu-system-ingress"
```

If you want to apply any additional configurations, add the values to your `values.yaml` file and reapply it using the `tanzu package installed update` command:

`tanzu package installed update cloud-native-runtimes -p cnr.tanzu.vmware.com -v 2.3.1 --values-file values-issuer.yaml -n tap-packages`

## Verify Knative Serving Functionalities

Now having Knative Serving up and running, the fun can begin :grin:. I'm using a simple web-server application for my validation. I created the Knative Service imperatively: `kn service create containerdays23-v1 --image registry.cloud-garage.net/rguske/webapp:v1 -n myapp --port 80`

Again, I will cover Serving in detail in my next post of this series. For now, I keep it at a basic level. The Knative Service deployment can be checked using `kn service list -n myapp`

```shell
kn service list -n myapp

NAME                 URL                                                                          LATEST                     AGE     CONDITIONS   READY     REASON
containerdays23-v2   https://containerdays23-v1.myapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net   containerdays23-v1-00001   5m44s   1 OK / 3     Unknown   CertificateNotReady : Certificate route-a192b381-d677-40ea-8e49-6287601ed5ba is not ready.
```

{{< admonition bug "CertificateNotReady" true >}}
CertificateNotReady : Certificate route-a192b381-d677-40ea-8e49-6287601ed5ba is not ready.
{{< /admonition >}}

This wasn't expected :thinking:.

There's a resource in Knative named `certificates.networking.internal.knative.dev`. I checked the state of it in my `myapp` namespace.

```shell
k -n myapp get certificates.networking.internal.knative.dev route-a192b381-d677-40ea-8e49-6287601ed5ba -oyaml
```

And the status `message` surprised me very much.

```shell
[...]

status:
  conditions:
  - lastTransitionTime: "2023-10-03T15:38:23Z"
    message: 'error creating Certmanager Certificate: cannot create valid length CommonName:
      (containerdays23-v1.myapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net) still longer
      than 63 characters, cannot shorten'
    reason: CommonName Too Long
    status: "False"
    type: Ready
```

The issue occurs due the fact, that I kept the default configuration for Knative Service naming which is `"{{.Name}}.{{.Namespace}}.{{.Domain}}`. Therefore, the created name `containerdays23-v1.myapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net` is 4 characters too long :facepalm:

```shell
echo 'containerdays23-v1.myapp.jarvis.cpod-nsxv8.az-stc.cloud-garage.net' | wc -c

67
```

I looked it up and found that this is a known issue: [When using auto-tls, Knative Service Fails with CertificateNotReady.](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/troubleshooting.html#when-using-autotls-knative-service-fails-with-certificatenotready-19)

The solution for me was simply to shorten the `domain_template` by applying the following configuration:

| Key | Type |
| :-: | :-: |
| `domain_template` | "{{.Name}}.{{.Domain}}" |

Therefore, it is now: `containerdays23-v1.jarvis.cpod-nsxv8.az-stc.cloud-garage.net`

## Installing Knative Eventing :star:

Last but absolutely not least is the installation of the Knative Eventing package. Knative Eventing depends on Knative Serving. For the installation itself are no additional values necessary. Therefore, it's just straight forward.

```shell
tanzu package install eventing \
-p eventing.tanzu.vmware.com \
-v 2.2.4 \
-n tap-packages \
--poll-timeout 30m
```

Unfortunately, the Pod Security Admission Controllers hits the second time.

{{< admonition warning "forbidden: violates PodSecurity 'restricted:latest'" true >}}
[...]

message: 'pods "rabbitmq-broker-webhook-7557658cf6-9qxpw" is forbidden: violates
      PodSecurity "restricted:latest": unrestricted capabilities (container "rabbitmq-broker-webhook"
      must set securityContext.capabilities.drop=["ALL"]), seccompProfile (pod or
      container "rabbitmq-broker-webhook" must set securityContext.seccompProfile.type
      to "RuntimeDefault" or "Localhost")'

[...]
{{< /admonition >}}

This is due to the PSA "default baseline" in vSphere with Tanzu (TKGS). I had to apply the PSA `pod-security.kubernetes.io/enforce=privileged`, like [here](#pod-security-admission-controller-punch), to the new created namespaces for Eventing during the installation of the package as well. You have to be "quick".

```shell
kubectl label --overwrite ns --all pod-security.kubernetes.io/enforce=privileged

namespace/cert-manager not labeled
namespace/default not labeled
namespace/knative-eventing labeled
namespace/knative-serving labeled
namespace/knative-sources not labeled
namespace/kube-node-lease not labeled
namespace/kube-public not labeled
namespace/kube-system labeled
namespace/myapp not labeled
namespace/secretgen-controller labeled
namespace/tanzu-system-ingress not labeled
namespace/tap-packages not labeled
namespace/tkg-system labeled
namespace/triggermesh labeled
namespace/vmware-sources not labeled
namespace/vmware-system-antrea not labeled
namespace/vmware-system-auth not labeled
namespace/vmware-system-cloud-provider labeled
namespace/vmware-system-csi not labeled
namespace/vmware-system-tkg not labeled
namespace/vmware-system-tmc not labeled
```

I wonder if this behaviour survives user-feedback...

However, this is treated differently in Tanzu Kubernetes Grid (M).

## Conclusion

Knative provides a more approachable abstraction on top of Kubernetes. I will dive deeper into its comprehensive feature set in the next blog posts of this series. Cloud Native Runtimes (CNR) is VMware's official Knative offering which is part of the modular application development platform Tanzu Application Platform (TAP). The streamlined installation of Knative Serving and Eventing is done using Tanzu Packages for TAP. Tanzu Packages are applications which are not only packaged but also completely life-cycled in a modern way using the VMware maintained OSS project Carvel.

The next blog post will show a lot of the great features which are provided by Knative Serving. Buckle up for some cool demos.

## Resources

- [Knative project](https://knative.dev/docs/)
- [Demoing Knative Serving and Eventing using Demo-Magic-Scripts](https://rguske.github.io/post/demoing-knative-serving-and-eventing-using-demo-magic-scripts/)
- [Serving](https://knative.dev/docs/serving/)
- [Eventing](https://knative.dev/docs/eventing/)
- [Functions](https://knative.dev/docs/functions/)
- [Knative Quickstart](https://knative.dev/docs/install/quickstart-install/)
- [KinD](https://github.com/kubernetes-sigs/kind)
- [Tanzu Cloud Native Runtimes](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.3/tanzu-cloud-native-runtimes/cnr-overview.html)
- [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html)
- [Prerequisites for installing Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)
- [Resource requirements](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)
- [Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.6/cluster-essentials/deploy.html)
- [Packages](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2/about-tkg/packages-index.html)
- [Sharing Container Images using the Docker CLI or Carvel's imgpkg](https://rguske.github.io/post/sharing-container-images-using-the-docker-cli-or-carvels-imgpkg/)
- [Deploy VMware Tanzu Packages from a private Container Registry](https://rguske.github.io/post/deploy-tanzu-packages-from-a-private-registry/#optional-kapp-controller-secret-vs-configmap)
- [secretgen-controller](https://github.com/carvel-dev/secretgen-controller)
- [shared ingress issuer](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/security-and-compliance-tls-and-certificates-ingress-issuer.html)
- [Configure Pod Security for TKR 1.25 and Later](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-B57DA879-89FD-4C34-8ADB-B21CB3AE67F6.html)

