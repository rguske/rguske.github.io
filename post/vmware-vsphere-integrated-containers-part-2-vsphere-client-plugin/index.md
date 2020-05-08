# vSphere Integrated Containers Part II: vSphere Client Plug-In


In this section, I´d like to give you a walkthrough on how to install the VIC vSphere Client Plug-In for the vCenter Server Appliance. I´m already using the vCenter Server Appliance aka vCSA in Version 6.7.0. In general, it is important to know that a minimum version of vCenter Server 6.5.0d is required to make use of the plug-in.

However! It is also worth to be mentioned that the plug-in is not necessary or a requirement to run VIC as well as for the deployment of a Virtual Container Host aka **VCH**. But it makes the initial deployment of a VCH easier at the first beginning until you are more and more familiar with the vic-machine utility.
The first step from the installation of the vSphere Client Plug-In is to download the vic-machine-bundle from the VIC-Appliance by using <a href="https://en.wikipedia.org/wiki/CURL" target="_blank">`curl`</a>.

To set up the necessary commands from within the vCSA we first have to enable SSH over the Virtual Appliance Management Interface aka VAMI by using port 5480. By default SSH is disabled.

https://lab-vcsa67-001:5480

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092710.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092710.jpg" class="center" width="800" >}}

Toggle the switch to enable SSH Login on the vCSA

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092734.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092734.jpg" class="center" width="800" >}}

But wait! To make sure if everything went right after the installation, we should check before and after. And how can we check if there isn´t a vic plug-in before the installation and afterwards it gets shown? Correct - over the Managed Object Browser aka MOB.

https://lab-vcsa67-001/mob

You won´t find anything declared with vic- under *content/ ExtensionManager/ extensionList*.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_095438.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_095438.jpg" class="center" width="800" >}}

Now let's establish a ssh-connection to the vCSA.

```shell
ssh root@192.168.100.72
```

Write and execute:

```shell
shell
```

After the login we now want to download and unpack the vic-bundle tar-file from the VIC Appliance to start the installation of the VIC vSphere Client Plug-In. You should just change my IP-Address to yours in the following lines.

```shell
export VIC_ADDRESS=192.168.100.160
export VIC_BUNDLE=vic_v1.4.0.tar.gz
curl -kL https://${VIC_ADDRESS}:9443/files/${VIC_BUNDLE} -o ${VIC_BUNDLE}
```

```shell
tar -zxf ${VIC_BUNDLE}
cd vic/ui/VCSA
```

By using the Linux command `ls` (list)`-ltr` (sort by change date) you should see the downloaded file.
The next and final step before the installation of the vCenter plug-in can begin, is the execution of the installation script.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_094828.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_094828.jpg" class="center" width="800" >}}

Execute the install.sh script by entering:

```shell
./install.sh
```

...and provide the requested data (vCSA FQDN as well as vCenter Server Administrator Credentials). By the end it should look like this:

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_095702.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_095702.jpg" class="center" width="800" >}}

Now let´s check again the extensionList over the vCenter Managed Object Manager - MOB.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180707_090912.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180707_090912.jpg" class="center" width="800" >}}

Voila!
Okay, now we are close to make use of the vic plug-in capabilities in our vSphere-WebClient but before we can go ahead, we need to restart the H5-Client as well as the Flex-based vSphere Web Client by running the following commands:

```shell
service-control --stop vsphere-ui && service-control --start vsphere-ui
service-control --stop vsphere-client && service-control --start vsphere-client
```

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100412.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100412.jpg" class="center" width="800" >}}

Because we want to hold our vCSA as clean as a Appliance should be, we have to get rid of our tracks by starting the cleaning process through the deletion of the unpacked tar-file.

```shell
rm *.tar.gz
rm -R vic
```

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100743.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100743.jpg" class="center" width="800" >}}


**Disable** SSH when you are finished!

The VIC plug-in should now be available under *Menu/ vSphere Integrated Containers*.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092711.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_092711.jpg" class="center" width="300" >}}

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_101449.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_101449.jpg" class="center" width="800" >}}

Now we have established the **Integration** into **vSphere** but what about the **Containers**? It won´t last long...

---

**Continue with:**

<a href="/post/vmware-vsphere-integrated-containers-part-3-deployment-of-a-virtual-container-host/">**vSphere Integrated Containers Part III: Deployment of a Virtual Container Host**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-4-docker-run-a-container-vm/">**vSphere Integrated Containers Part IV: docker run a Container-VM**</a>

---

**Previous articles:**

<a href="/post/vmware-vsphere-integrated-containers-part-1-ova-deployment/">**vSphere Integrated Containers Part I: OVA Deployment**</a>

<a href="/post/vmware-vsphere-integrated-containers-introduction/">**vSphere Integrated Containers: Introduction**</a>

---

**<center>Thanks for reading!</center>**
