# Deploy VMware Tanzu Packages from a private Container Registry


## Offline Requirements

During a recent Tanzu engagement, my customer wanted to make use of the VMware [Tanzu Packages](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-packages-index.html) to extend the core functionalities of their existing Kubernetes clusters. As a requirement, an offline deployment of the Packages was raised, which in other words means, deployments from their own private Container Registry. The intention behind it is simple.

All of the Tanzu Packages shown in *Table I* below are provided by VMware. VMware is ensuring that the images for the Packages are stable, secure and compliant. But in order to deploy a Package, we need unristricted internet acceess to `*.projects.registry.vmware.com`. To bypass this "online" requirement, deploying the Packages from a private Container Registry like [Harbor](https://goharbor.io) for example, is a typical use case for most clients.

*Table I: List of User-Managed Packages*

| Function | Package Package | Repository |
| :--: | :--: | :--: |
| Certificate management | cert-manager | tanzu-standard |
| Container networking | multus-cni | tanzu-standard |
| Container registry |harbor |tanzu-standard |
| Ingress control | contour | tanzu-standard |
| Log forwarding | fluent-bit | tanzu-standard |
| Monitoring | grafana | tanzu-standard |
| Monitoring | prometheus | tanzu-standard |
| Service discovery | external-dns | tanzu-standard |

{{< admonition info "Repository URL" true >}}
Table column *Repository*, is the name of the package repository (`tanzu-standard`) which is configured by default in namespace `tanzu-package-repo-global` and which is normally configured with URL `projects-stg.registry.vmware.com/tkg/packages/standard/repo`.
{{< /admonition >}}

By following the steps provided in this post, you will be able to deploy the kapp-controller as well as all Tanzu Packages from your own private registry onto your eagerly waiting Kubernetes clusters. We start with some prerequisites.

## Prerequisites

We have to equip our `shell` with some additional CLI's in order to proceed further.

### Tanzu Cli

The `tanzu` CLI ships alongside a compatible version of the `kubectl` CLI. Download the CLI(s) from VMware's Customer Connect portal. 

- [Download - Tanzu CLI](https://customerconnect.vmware.com/en/downloads/details?downloadGroup=TKG-141&productId=988&rPId=82536)

If you are new to the topic, check out the official documentation to get even more details.

- [Documentation - Tanzu CLI](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-install-cli.html)

In short for Linux (MacOS and Windows instructions via Docs):

```shell
# unpack the tar ball
tar -xvf tanzu-cli-bundle-linux-amd64.tar
```

```shell
# unpacked tar ball
.
├── cluster
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-cluster-linux_amd64
├── core
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-core-linux_amd64
├── imgpkg-linux-amd64-v0.10.0+vmware.1.gz
├── kapp-linux-amd64-v0.37.0+vmware.1.gz
├── kbld-linux-amd64-v0.30.0+vmware.1.gz
├── kubernetes-release
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-kubernetes-release-linux_amd64
├── login
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-login-linux_amd64
├── management-cluster
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-management-cluster-linux_amd64
├── manifest.yaml
├── package
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-package-linux_amd64
├── pinniped-auth
│   ├── plugin.yaml
│   └── v1.4.1
│       └── tanzu-pinniped-auth-linux_amd64
├── vendir-linux-amd64-v0.21.1+vmware.1.gz
└── ytt-linux-amd64-v0.34.0+vmware.1.gz
```

```shell
# install the tanzu cli
sudo install cli/core/v1.4.0/tanzu-core-linux_amd64 /usr/local/bin/tanzu

# version check
tanzu version

# tanzu cli update
tanzu update

# remove existing plugins from any previous CLI installations
tanzu plugin clean

# install all the plugins for the specific TKG release
tanzu plugin install --local cli all

# check plugin installation status
tanzu plugin list
```

### kubectl

The compatible `kubectl` version can be downloaded via the provided link as well. 

Just to have it mentioned - You will notice that my `code` examples in this post will often has just a `k` instead of `kubectl` when I used the CLI. This is because I'm always using aliases for certain CLI's in my shell. `alias k=kubectl` 

```shell
# unpack the binary
gzip -d kubectl-linux-v1.21.2+vmware.1.gz

# make it executable
chmod +x kubectl-linux-v1.21.2+vmware.1

# move the executable to your /usr/local/bin
sudo mv kubectl-linux-v1.21.2+vmware.1 /usr/local/bin/kubectl

# version check
kubectl version
```

### imgpkg

The [Carvel](https://carvel.dev) tool `imgpkg`, pronounced "image package", is a tool that allows you to store a set of arbitrary files as an OCI image and is also being used for e.g. deploying Tanzu Kubernetes Grid in Internet-restricted environments and when building your own machine images.

```shell
# change into the cli folder of the unpacked Tanzu cli
cd cli

# unpack the Gnu zipped archive
gunzip imgpkg-linux-amd64-v0.10.0+vmware.1.gz

# make it executable
chmod ugo+x imgpkg-linux-amd64-v0.10.0+vmware.1

# move the binary into /usr/local/bin
mv ./imgpkg-linux-amd64-v0.10.0+vmware.1 /usr/local/bin/imgpkg
```

### Create a new Public Project in Harbor

Create a new project/repository in your registry to which we can `push` (upload) the packages (images) to. I've created a new one called `tanzu-packages` in my Harbor registry and marked it as `Public Repository`. I will come to this later again why I configured it as a public repository.

{{< image src="/img/posts/202202_tanzu_packages_offline/rguske_post_empty_harbor_repo.png" caption="Figure I: New public Harbor Project" src-s="/img/posts/202202_tanzu_packages_offline/rguske_post_empty_harbor_repo.png" >}}

I've written a couple of blog posts covering e.g. the deployment by using `helm` or Tanzu Mission Control Catalog as well as for configuring specific features of the Harbor Container Registry.

Check out:

:memo: [Deploying Tanzu Packages using Tanzu Mission Control Catalog](https://rguske.github.io/post/deploying-tanzu-packages-using-tanzu-mission-control-catalog/)

:memo: [vSphere 7 with Kubernetes supercharged - Helm, Harbor and Tanzu Kubernetes Grid](https://rguske.github.io/post/vsphere-7-with-kubernetes-supercharged-helm-harbor-tkg/)

:memo: [Using Harbor with the VMware Event Broker Appliance](https://rguske.github.io/post/using-harbor-with-the-vcenter-event-broker-appliance/)

### Download the Harbor Certificate

You can skip this section if you are using a different registry. 

> Please have the certificate of your registry by hand because we need it a few times more.

Download the Harbor certificate and add it (if not already done) to your Docker config as well.

```shell
# enter the FQDN of your Harbor instance (e.g. harbor.jarvis.tanzu)
read REGISTRY

# download the certificate
sudo wget -O ~/Downloads/ca.crt https://$REGISTRY/api/v2.0/systeminfo/getcert --no-check-certificate
```

Preparations for a `docker login`:

```shell
# create a folder named like your registry
sudo mkdir -p /etc/docker/certs.d/$REGISTRY

# download the certificate
sudo wget -O /etc/docker/certs.d/$REGISTRY/ca.crt https://$REGISTRY/api/v2.0/systeminfo/getcert --no-check-certificate

# restart docker daemon
systemctl restart docker

# login
docker login harbor.jarvis.tanzu

Username: admin
Password:
Login Succeeded
```

## `pull` and `push` the Kapp-Controller Image

The first component which we make offline (offline == private registry) available is the [kapp-controller](https://carvel.dev/kapp-controller/) in VMware's own maintained version (`v0.23.0_vmware.1`).

{{< admonition info "kapp-controller" true >}}
kapp-controller's declarative APIs and layered approach enables you to build, deploy, and manage your own applications.

*Source*: https://carvel.dev/kapp-controller/
{{< /admonition >}}

```shell
# pull the kapp-controller image locally first
docker pull projects.registry.vmware.com/tkg/kapp-controller:v0.23.0_vmware.1

# tag the kapp-controller image to be prepared for the image push
docker tag projects.registry.vmware.com/tkg/kapp-controller:v0.23.0_vmware.1 harbor.jarvis.tanzu/tanzu-packages/kapp-controller:v0.23.0_vmware.1
```

Locally it should look like the following:

```shell
docker images

REPOSITORY                                           TAG                IMAGE ID       CREATED        SIZE
harbor.jarvis.tanzu/tanzu-packages/kapp-controller   v0.23.0_vmware.1   0076b17e8c71   5 months ago   639MB
projects.registry.vmware.com/tkg/kapp-controller     v0.23.0_vmware.1   0076b17e8c71   5 months ago   639MB
```

Push it:

```shell
# push the image to the destination registry
docker push harbor.jarvis.tanzu/tanzu-packages/kapp-controller:v0.23.0_vmware.1
```

{{< image src="/img/posts/202202_tanzu_packages_offline/rguske_post_kapp_push_harbor.png" caption="Figure II: Pushed kapp-controller image" src-s="/img/posts/202202_tanzu_packages_offline/rguske_post_kapp_push_harbor.png" >}}

## Install the Kapp-Controller

In order to install the kapp-controller to our Kubernetes cluster, a deployment manifest file has to be created first. Create a new file called e.g. `kapp-controller.yaml` and add the provided specifications from the [docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-packages-prep-tkgs-kapp.html#kapp-controller).

Open the file and jump to line number `1278`. By using `vim` for example, you can enable line numbers by executing `:set numbers` and jumping to the line by entering `:1278`. Otherwise, simply search `/` for `image: projects`.

Replace the original repository and image with yours.

```yaml
# replacing the old repository and image with the private one
[...]

1278         image: harbor.jarvis.tanzu/tanzu-packages/kapp-controller:v0.23.0_vmware.1
1279         name: kapp-controller
1280         ports:

[...]
```

Install the kapp-controller by executing `k apply -f kapp-controller.yaml`.

```shell
k apply -f kapp-controller.yaml

namespace/tkg-system created
namespace/tanzu-package-repo-global created
apiservice.apiregistration.k8s.io/v1alpha1.data.packaging.carvel.dev created
service/packaging-api created
customresourcedefinition.apiextensions.k8s.io/internalpackagemetadatas.internal.packaging.carvel.dev created
customresourcedefinition.apiextensions.k8s.io/internalpackages.internal.packaging.carvel.dev created
customresourcedefinition.apiextensions.k8s.io/apps.kappctrl.k14s.io created
customresourcedefinition.apiextensions.k8s.io/packageinstalls.packaging.carvel.dev created
customresourcedefinition.apiextensions.k8s.io/packagerepositories.packaging.carvel.dev created
configmap/kapp-controller-config created
deployment.apps/kapp-controller created
serviceaccount/kapp-controller-sa created
clusterrole.rbac.authorization.k8s.io/kapp-controller-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/kapp-controller-cluster-role-binding created
clusterrolebinding.rbac.authorization.k8s.io/pkg-apiserver:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/pkgserver-auth-reader created
```

> Notice: The Kubernetes namespaces `tkg-system` as well as `tanzu-package-repo-global` was created as part of the deployment. Keep this in mind!

Installation successfully :white_check_mark:

```shell
k -n tkg-system get deploy,po,configmap,replicasets.apps
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kapp-controller   1/1     1            1           22h

NAME                                   READY   STATUS    RESTARTS   AGE
pod/kapp-controller-5fd59df9dd-xmvmj   1/1     Running   0          21h

NAME                               DATA   AGE
configmap/kapp-controller-config   5      22h
configmap/kube-root-ca.crt         1      22h

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/kapp-controller-5fd59df9dd   1         1         1       22h
```

## Making Packages offline available

Since our goal is to deploy packages from our private registry, we have to make the package images offline available first. Downloading, bundling and uploading the images can easily be performed with the following two `imgpkg` commands:

{{< admonition warning "Registry Certificate" true >}}
The registry certificate is needed to upload the image bundle (`--registry-ca-cert-path=ca.crt`).
{{< /admonition >}}

```shell
# use the imgpkg copy command to download the packages and to pack them into a tar ball
imgpkg copy -b projects.registry.vmware.com/tkg/packages/standard/repo:v1.4.0 --to-tar ~/Downloads/packages/packages.tar

# use the imgpkg copy command to `push` the content into your private container registry
imgpkg copy --tar packages.tar --to-repo harbor.jarvis.tanzu/tanzu-packages/tanzu-packages --registry-ca-cert-path=ca.crt --registry-username=admin --registry-password=$PASSWORD
```

:movie_camera: Watch the thrilling and somewhat boring recording: (:fast_forward: is your friend :wink:)

<script id="asciicast-wKfybXtagW70j8Q62wD8D7UfV" src="https://asciinema.org/a/wKfybXtagW70j8Q62wD8D7UfV.js" async></script>

{{< image src="/img/posts/202202_tanzu_packages_offline/rguske_post_packages_bundle_uploaded.png" caption="Figure III: Pushed kapp-controller image" src-s="/img/posts/202202_tanzu_packages_offline/rguske_post_packages_bundle_uploaded.png" >}}

## Update the Kapp-Controller Configmap

Before we are able to announce the new repository to the kapp-controller (`tanzu package repository add`) we have to make the kapp-controller trust our private registry by providing the certificate. In order to make this happen, the kapp-controller `configMap` needs an update.

**OTHERWISE** you will quickly see this error message:

```shell
tanzu package repository add tanzu-packages-offline --url harbor.jarvis.tanzu/tanzu-packages/tanzu-packages:v1.4.0 -n tanzu-package-repo-global

- Adding package repository 'tanzu-packages-offline'

| Getting package repository 'tanzu-packages-offline'
| Validating provided settings for the package repository
| Updating package repository resource
- Waiting for 'PackageRepository' reconciliation for 'tanzu-packages-offline'


Error: resource reconciliation failed: Error: Syncing directory '0': Syncing directory '.' with imgpkgBundle contents: Imgpkg: exit status 1 (stderr: Error: Checking if image is bundle: Collecting images: Working with harbor.jarvis.tanzu/tanzu-packages/tanzu-packages:v1.4.0: Get "https://harbor.jarvis.tanzu/v2/": x509: certificate signed by unknown authority
)
. Reconcile failed: Fetching resources: Error (see .status.usefulErrorMessage for details)
Error: exit status 1

✖  exit status 1
```

{{< admonition failure "Certificate Error" true >}}
x509: certificate signed by unknown authority
{{< /admonition >}}

And by checking the reconciliation status of the repository you will see:

```shell
tanzu package repository list -A

| Retrieving repositories...
  NAME                 REPOSITORY                                         TAG     STATUS                   DETAILS                  NAMESPACE
  company-repo-jarvis  harbor.jarvis.tanzu/tanzu-packages/tanzu-packages  v1.4.0  Reconcile failed: Fe...  Error: Syncing direc...  tanzu-packages
```

Certificates... :roll_eyes:

Let's get it done:

**Step 1:** Edit the kapp-controller `configMap`:

```shell
# edit configMap
k -n tkg-system edit cm kapp-controller-config
```

**Step 2:** Add your certificate data under section `caCerts` like in my example shown below:

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIDWzCCAkOgAwIBAgIRAMODWpLzIWy3JocU4JBtIrIwDQYJKoZIhvcNAQELBQAw
    LTEXMBUGA1UEChMOUHJvamVjdCBIYXJib3IxEjAQBgNVBAMTCUhhcmJvciBDQTAe
    Fw0yMjAxMTExMDEwMDlaFw0zMjAxMDkxMDEwMDlaMC0xFzAVBgNVBAoTDlByb2pl
    Y3QgSGFyYm9yMRIwEAYDVQQDEwlIYXJib3IgQ0EwggEiMA0GCSqGSIb3DQEBAQUA

    [...]
    -----END CERTIFICATE-----
  dangerousSkipTLSVerify: ""
  httpProxy: ""
  httpsProxy: ""
  noProxy: ""
kind: ConfigMap
metadata:
  annotations:
    kapp.k14s.io/change-group: apps.kappctrl.k14s.io/kapp-controller-config
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"caCerts":"","dangerousSkipTLSVerify":"","httpProxy":"","httpsProxy":"","noProxy":""},"kind":"ConfigMap","metadata":{"annotations":{"kapp.k14s.io/change-group":"apps.kappctrl.k14s.io/kapp-controller-config"},"name":"kapp-controller-config","namespace":"tkg-system"}}
  creationTimestamp: "2022-02-08T12:31:35Z"
  name: kapp-controller-config
  namespace: tkg-system
  resourceVersion: "1168617"
  uid: 73e9cc55-02d4-4790-a7ee-eaa38b15c894
```

Save your adjustments `:wq`.

**Step 3:** Restart/`delete` the kapp-controller pod to take effect changes:

```shell
# delete the kapp-controller
k -n tkg-system delete pod kapp-controller-5fd59df9dd-xmvmj
```

**Step 4:** Add the URL of our private repository and use the namespace `tanzu-package-repo-global`:

```shell
# add the offline package repository
tanzu package repository add tanzu-packages-offline --url harbor.jarvis.tanzu/tanzu-packages/tanzu-packages:v1.4.0 -n tanzu-package-repo-global
```

{{< admonition info "Different Namespace" true >}}
If you choose a different namespace, the Packages will only be visible and deployable in that particular namespace.

*Source:* beyondelastic.com - [Tanzu Packages Explained](https://beyondelastic.com/2022/01/04/tanzu-packages-explained/)
{{< /admonition >}}

**Step 5:** Validate that the kapp-controller can successfully `reconcile` the repository:

```shell
# validate the configuration of the repository
tanzu package repository list -n tanzu-package-repo-global

- Retrieving repositories... 
  NAME                    REPOSITORY                                         TAG     STATUS               DETAILS  
  tanzu-packages-offline  harbor.jarvis.tanzu/tanzu-packages/tanzu-packages  v1.4.0  Reconcile succeeded 
```

**Step 6:** Also, validate the available (offline) packages:

```shell
# check package availability
tanzu package available list -n tanzu-package-repo-global

- Retrieving available packages...
  NAME                           DISPLAY-NAME  SHORT-DESCRIPTION                                                                                           LATEST-VERSION
  cert-manager.tanzu.vmware.com  cert-manager  Certificate management                                                                                      1.1.0+vmware.1-tkg.2
  contour.tanzu.vmware.com       Contour       An ingress controller                                                                                       1.17.1+vmware.1-tkg.1
  external-dns.tanzu.vmware.com  external-dns  This package provides DNS synchronization functionality.                                                    0.8.0+vmware.1-tkg.1
  fluent-bit.tanzu.vmware.com    fluent-bit    Fluent Bit is a fast Log Processor and Forwarder                                                            1.7.5+vmware.1-tkg.1
  grafana.tanzu.vmware.com       grafana       Visualization and analytics software                                                                        7.5.7+vmware.1-tkg.1
  harbor.tanzu.vmware.com        Harbor        OCI Registry                                                                                                2.2.3+vmware.1-tkg.1
  multus-cni.tanzu.vmware.com    multus-cni    This package provides the ability for enabling attaching multiple network interfaces to pods in Kubernetes  3.7.1+vmware.1-tkg.1
  prometheus.tanzu.vmware.com    prometheus    A time series database for your metrics                                                                     2.27.0+vmware.1-tkg.1
```

## Deploy Package Cert-Manager

Now that we have our private repository configured, let's try deploying a first Tanzu Package to our Kubernetes cluster.

```shell
# list cert-manager package availability
tanzu package available list cert-manager.tanzu.vmware.com -n tanzu-package-repo-global

# check a specific cert-manager version
tanzu package available get cert-manager.tanzu.vmware.com/1.1.0+vmware.1-tkg.2 -n tanzu-package-repo-global

# install cert-manager version
tanzu package install cert-manager -p cert-manager.tanzu.vmware.com -v 1.1.0+vmware.1-tkg.2 -n tanzu-package-repo-global
```

The deployment process is going to notify you when the installation of the Tanzu Package `cert-manager.tanzu.vmware.com` has finished.

```shell
[...]
- Waiting for 'PackageInstall' reconciliation for 'cert-manager'
/ 'PackageInstall' resource install status: Reconciling

 Added installed package 'cert-manager'
```

:tada: :tada: :tada:

The reconciliation status of a specific Package can be verified every time using:

```shell
tanzu package installed list -n tanzu-package-repo-global

- Retrieving installed packages...
  NAME          PACKAGE-NAME                   PACKAGE-VERSION       STATUS
  cert-manager  cert-manager.tanzu.vmware.com  1.1.0+vmware.1-tkg.2  Reconcile succeeded
```

## OPTIONAL: kapp-controller secret vs. configMap

Instead of configuring the `configMap` of the kapp-controller, a creation of a Kubernetes `secret` can be used as an alternative. I validated both ways successfully. A more detailed description can be found on the official Carvel documentation here: [Configuring the Controller](https://carvel.dev/kapp-controller/docs/v0.32.0/controller-config/#controller-configuration-spec)

Here's my example I've tested:

```yaml
apiVersion: v1
kind: Secret
metadata:
  # Name must be `kapp-controller-config` for kapp controller to pick it up
  name: kapp-controller-config
  # Namespace must match the namespace kapp-controller is deployed to
  namespace: tkg-system
stringData:
  # A cert chain of trusted ca certs. These will be added to the system-wide
  # cert pool of trusted ca's (optional)
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIDWzCCAkOgAwIBAgIRAMODWpLzIWy3JocU4JBtIrIwDQYJKoZIhvcNAQELBQAw

    [...]
    -----END CERTIFICATE-----
  # The url/ip of a proxy for kapp controller to use when making network
  # requests (optional)
  httpProxy: ""
  # The url/ip of a tls capable proxy for kapp controller to use when
  # making network requests (optional)
  httpsProxy: ""
  # A comma delimited list of domain names which kapp controller should
  # bypass the proxy for when making requests (optional)
  noProxy: ""
  # A comma delimited list of domain names for which kapp controller, when
  # fetching images or imgpkgBundles, will skip TLS verification. (optional)
  dangerousSkipTLSVerify: ""
```

## Resources

- VMware Docs - [Install the Tanzu CLI and Other Tools](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-install-cli.html)
- VMware Docs - [Install and Configure Packages](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-packages-index.html)
- beyondelastic.com - [Tanzu Packages Explained](https://beyondelastic.com/2022/01/04/tanzu-packages-explained/)
- beyondelastic.com - [Configure vSphere with Tanzu behind a Proxy plus TKG Extensions](https://beyondelastic.com/2021/06/21/configure-vsphere-with-tanzu-behind-a-proxy-plus-tkg-extensions/)
- Stay here :wink: - [Deploying Tanzu Packages using Tanzu Mission Control Catalog](https://rguske.github.io/post/deploying-tanzu-packages-using-tanzu-mission-control-catalog/)
- Carvel Docs - [Configuring the Controller](https://carvel.dev/kapp-controller/docs/v0.32.0/controller-config/#controller-configuration-spec)
- Carvel Docs - [Authenticating to Private Registries](https://carvel.dev/kapp-controller/docs/v0.32.0/private-registry-auth/)
