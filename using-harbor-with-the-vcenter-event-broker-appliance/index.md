# Using Harbor with the vCenter Event Broker Appliance


The <a href="https://github.com/vmware-samples/vcenter-event-broker-appliance" target="_blank">vCenter Event Broker Appliance (VEBA)</a> is still one of my favorite open source projects these days and it is evolving rapidly and continuously through the great work of the two main contributors <a href="https://twitter.com/embano1" target="_blank">Michael Gasch</a> and <a href="https://twitter.com/lamw" target="_blank">William Lam</a> as well as through the valuable ***feedback*** from the community. I´m very proud to be part of the "inner circle" of folks who meet on a regular basis to discuss everything <a href="https://twitter.com/search?q=%23vcenter-event-broker-appliance&src=typed_query" target="_blank">#VEBA</a> and the keyword here is also ***feedback***.

Version 0.3 was recently with big updates announced which can be viewed via the following link: <a href="https://www.virtuallyghetto.com/2020/03/big-updates-to-the-vcenter-event-broker-appliance-veba-fling.html" target="_blank">Big updates to the vCenter Event Broker Appliance (VEBA) Fling</a>

Just to name a few updates:

- *VMware Event Router implementation*
- *AWS EventBridge support*
- *New event playload structure - cloudevents.io*

---

## Pulling images from Harbor fails

I was still testing the new version in my lab but this time I wanted to add another great open source project as a playmate to the round, the enterprise container image registry <a href="https://goharbor.io" target="_blank">Harbor</a>. VEBA ist using <a href="https://www.openfaas.com/" target="_blank">OpenFaaS®</a> as the built-in event processor and you define the function configuration in a YAML file, the `stack.yml`.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_120502.jpg" caption="Figure I: Original repository, image name and tag" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_120502.jpg" class="center" width="800" height="800" >}}

A closer look into the `stack.yml` file shows us that the official VMware repository is used. We only see the repository name ***vmware/***, the image name ***veba-powercli-tagging*** as well as the tag ***:latest***. This means that when no registry is configured at this point, the Docker default applies, which is then <a href="https://hub.docker.com/" target="_blank">Docker Hub</a>.

Instead of pulling the appropriate image(s) from Docker Hub, I wanted to use **Harbor** this time. In order to implement this, I have to replace the original image specification in the `stack.yml` and point to my Harbor instance, the corresponding project and the image with tag.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_120429.jpg" caption="Figure II: Modified repository, image name and tag" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_120429.jpg" class="center" width="800" height="800" >}}

---

## Create a project in Harbor

Before we continue with the deployment of a function and what I´ve observed when VEBA is trying to pull the images from Harbor, I´d like to give you a **short** intoduction "How to create a project in Harbor".

**1)** Login with *admin*, select *Projects* and give it a name.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_113744.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_113744.jpg" class="center" caption="Figure III: New project" >}}

**2)** Go to Members and add a new *User* to the project.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_113818.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_113818.jpg" class="center" width="800" height="800" caption="Figure IV: Add a user" >}}

**3)** The last step for now is to mark the project as *Public* so it is accessible to everyone and no `docker login` is required before. I also enable the *Automatically scan image on push* option, to let <a href="https://github.com/quay/clair" target="_blank">Clair</a> immediately scan images when they are pushed.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200322_061632.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200322_061632.jpg" class="center" caption="Figure V: Project configuration page" >}}

## Optional: Authentiction with LDAP

The following settings are not required but I wanted to authenticate Active-Directory users and groups to my project(s). I think it could be helpful to see a working configuration.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_044947.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_044947.jpg" class="center" caption="Figure VI: Harbor LDAP Authentication configuration" >}}

If everything is configured correctly, you should be able to add one or more AD groups to your project.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_044925.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_044925.jpg" class="center" caption="Figure VII: Harbor LDAP Authentication configuration" >}}

---

## Push an image to Harbor

For this demonstration I´m going to pick the <a href="https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/tagging" target="_blank">vSphere Tagging Function</a> example. Let´s create the *veba-powercli-tagging* image from the `Dockerfile` which is located in the following directory: ***vcenter-event-broker-appliance/examples/powercli/tagging/template/powercli/***.

We can build, tag and push the image in two ways:

### The Docker way - docker cli

Change into the directory were the `Dockerfile` is located, create a fresh new image out of it and directly assign it with a tag subsequently:

```Shell
$ cd ~/vcenter-event-broker-appliance/examples/powercli/tagging/template/powercli/

$ docker build -t harbor.jarvis.lab/veba/veba-powercli-tagging:latest

$ docker images
REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
harbor.jarvis.lab/veba/veba-powercli-tagging   latest              ab7d0277d326        15 seconds ago      368MB
```

Login into Harbor with an assigned user or a user of an assigned group:

```shell
$ docker login -u rguske harbor.jarvis.lab/veba
```

After a successful login, push the image to your project:

```shell
$ docker push harbor.jarvis.lab/veba/veba-powercli-tagging:latest
```

### The OpenFaaS way - faas-cli

The use of the <a href="https://docs.openfaas.com/cli/install/" target="_blank">faas-cli</a> makes the above described steps a little bit more comfortable and by the end, the `faas-cli` is using the native `docker` commands as well. To build as well as to tag the image, we just need to execute `faas-cli build -f stack.yml`. This command will pick the image specification from the `stack.yml` and will hand it over automatically.

Same result here:

```shell
REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
harbor.jarvis.lab/veba/veba-powercli-tagging   latest              3520bc9c8b85        32 seconds ago      368MB
```

You still need to login to Harbor if you want to execute the `push` command, otherwise you will receive the following error:

```shell
denied: requested access to the resource is denied
unauthorized: authentication required

2020/03/23 17:48:45 ERROR - Could not execute command: [docker push harbor.jarvis.lab/veba/veba-powercli-tagging:latest]
```

Login and execute `faas-cli push -f stack.yml`. The image will be pushed into your project.

{{< image src="/img/posts/202003_veba_harbor/CapturFiles-20200323_031023.jpg" src-s="/img/posts/202003_veba_harbor/CapturFiles-20200323_031023.jpg" class="center" caption="Figure VIII: Pushed image to the new project" >}}

---

## Deploy a function to VEBA

I don´t need to stress this topic in detail here because everything you need to be ***ready for take off*** :rocket: with VEBA is well documented on the Github page and if you miss something or need a more comprehensive insight on VEBA, visit this <a href="http://www.patrickkremer.com/veba/" target="_blank">blog series</a> by <a href="https://twitter.com/KremerPatrick" target="_blank">Patrick Kraemer</a>. Assuming the corresponding image is available in Harbor and the `stack.yml` is adapted accordingly (see *Figure II*), you can proceed with the steps provided on Github (<a href="https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/tagging" target="_blank">LINK</a>). Update the `stack.yml` and the `vc-tag-config.json` with your environment information.

I´m curious and therefore I established a `ssh` connection to my VEBA appliance. By executing `kubectl -n openfaas-fn get pods`, I can see how the deployment of the pod runs and the following is not what we want to see as a result:

```shell
NAME                            READY   STATUS             RESTARTS   AGE
powercli-tag-5ff88775fb-6g92k   0/1     ImagePullBackOff   0          28s

NAME                            READY   STATUS         RESTARTS   AGE
powercli-tag-5ff88775fb-6g92k   0/1     ErrImagePull   0          64s
```

I validated this behavior by executing `docker images` and observed that no image has been downloaded from Harbor. Troubleshooting often means, if an automatic process fails, go the manual way step by step. Consequently, the next step is to download the image manually.

```shell
docker pull harbor.jarvis.lab/veba/veba-powercli-tagging:latest
Error response from daemon: Get https://harbor.jarvis.lab/v2/: x509: certificate signed by unknown authority
```

**Interesting!**

{{< admonition failure "Failure" true >}}
"certificate signed by unknown authority"
{{< /admonition >}}

This is not based on the fact that I have not done a `docker login` before, as this is not necessary since we have made our project publicly available.

Following the official Docker documentation, this behavior is expected: <a href="https://docs.docker.com/engine/security/certificates/" target="_blank">Verify repository client with certificates</a>

To solve this issue, I´ve created a little script which downloads the root certificate from Harbor, creates the relevant directories, puts the certificate into them and restarts the docker service.

You can download the script <a href="https://github.com/rguske/using_harbor_with_veba" target="_blank">HERE</a> and copy it via `scp` into your VEBA appliance.

```shell
$ scp ~/Downloads/docker_harbor_cert.sh root@veba030:/
```

Make it executable with `chmod +x docker_harbor_cert.sh` and run it. The other option without using `scp` is, to establish a `ssh` connection and to copy and execute the neccessary lines directly.

## The script

```bash
#! /bin/bash

set -euo pipefail

# Ask for the Harbor FQDN
echo "Enter the FQDN (harbor.domain.com) of your Harbor registry:"

read REGISTRY

# Create folder for custom certificate as described in Docker docs https://docs.docker.com/engine/security/certificates/
mkdir -p /etc/docker/certs.d/$REGISTRY

# Download Registry Root Certificate
wget -O etc/docker/certs.d/$REGISTRY/ca.crt https://$REGISTRY/api/systeminfo/getcert --no-check-certificate

# Restart Docker service
systemctl restart docker
```

VEBA is now able to `pull` and run the image from Harbor.

```shell
$ kubectl -n openfaas-fn get pods

NAME                            READY   STATUS    RESTARTS   AGE
powercli-tag-5ff88775fb-wqqqf   1/1     Running   0          84s
```

---

<center>Thanks for reading.</center>
