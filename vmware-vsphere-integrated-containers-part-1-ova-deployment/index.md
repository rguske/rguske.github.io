# vSphere Integrated Containers Part I: OVA Deployment


## Environment Pre-Requisites:

- vSphere Enterprise Plus license or vSphere Remote Office Branch Office (ROBO) Advanced (!)
 * Dependency on the vDistributed Switch
 * VIC also supports VMware NSX
- User with administrative credentials to vCenter
- Internet Access for downloading images
- min. two vDistributed Switch Port Groups
 * for public communication (VCH to external network)
 * for inter containers communication create a dedicated port group for use as the bridge network for each VCH
 * If DHCP is not available on these segments, please, request a range of free IP-Addresses.

{{< admonition info "Info" true >}}
You´ll find all necessary pieces of information with regards to Licensing as well as Deployment Requirements on the official <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/vic_installation_prereqs.html" target="_blank">VIC Github Page</a>.
{{< /admonition >}}

## Deploying the OVA

To start with, we have to download the latest bits from myvmware.com here --> <a href="https://my.vmware.com/en/web/vmware/info/slug/datacenter_cloud_infrastructure/vmware_vsphere_integrated_containers/1_4" target="_blank">VIC Version 1.4.1</a>. After having downloaded the 3,12 GB ova-file we´ll start provisioning the VIC Virtual Appliance over the vSphere Web Client onto your vSphere Datacenter, Cluster or ESXi Host. I suppose that you are already familiar with these steps but if not, please go <a href="https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.vm_admin.doc/GUID-17BEDA21-43F6-41F4-8FB2-E01D275FE9B4.html" target="_blank">here</a> first.

If you´re already familiar with it and you´re more interested in automting an OVA deployment, I´d recommend reading this <a href="http://cloudmaniac.net/ova-ovf-deployment-using-govc-cli/" target="_blank"> Post </a> by <a href="https://twitter.com/woueb" target="_blank"> Romain Decker </a>.

### Pick the OVA file

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_104118.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_104118.jpg" class="center" width="800"  >}}

### Select a compute resource

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111348.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111348.jpg" class="center" width="800"  >}}

### Review details

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111747.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111747.jpg" class="center" width="800"  >}}

### License agreements

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111946.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_111946.jpg" class="center" width="800"  >}}

### Select storage

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112111.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112111.jpg" class="center" width="800"  >}}

### Select networks

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112335.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112335.jpg" class="center" width="800"  >}}

At this point, I´d like to stress out again to the VIC documentation regarding the <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/deploy_vic_appliance.html" target="_blank">use of SSH </a>. SSH is needed when you perform upgrades or the following:

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180619_084703.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180619_084703.jpg" class="center" width="800"  >}}

### Appliance Configuration

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112527.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_112527.jpg" class="center" width="800"  >}}

If you decide to use static IP-Addresses like me, please use spaces and not commas to separate multiple DNS-Servers.

### Network Properties

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113830.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113830.jpg" class="center" width="800"  >}}

I´ve also decided to create an example user through the wizard, which gets the username prefix you´ve chosen in point 5 in this section. I´m fine with the predetermined prefix *vic*.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113931.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113931.jpg" class="center" width="800"  >}}

### Ready to complete

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113956.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180607_113956.jpg" class="center" width="800"  >}}

Lean back and let the vCenter do its job... ... ...FINISHED!

{{< admonition tip "Tip" true >}}
If you think *"Hey this H5-Client Dark Theme looks very slick! Where I can toggle the switch?"* unfortunately one has to name, that this is not a feature in the vSphere H5-Client, it´s a Browser-Extension by Jens L. aka BeryJu and available for Chrome and Firefox. You´ll find him on <a href="https://github.com/BeryJu" target="_blank">Github</a> as well as on his <a href="https://beryju.org/en" target="_blank">Blog</a>. Thanks, Jens for the nice work.
{{< /admonition >}}

---

{{< admonition warning "Attention" true >}}
When you add the extension, VMware will not provide support when you´re facing issues!
And - I´d recommend using only browsers where the language is set to English! In other cases, you could hit the issue <a href="https://github.com/BeryJu/dark-vcenter/issues/36" target="_blank">Other browser language than English breaks CSS inject #36</a>
{{< /admonition >}}

---

**Here you´ll find the extensions.**

<a href="https://chrome.google.com/webstore/search/Dark%20vCenter" target="_blank">Dark-vCenter for Google Chrome</a>

<a href="https://addons.mozilla.org/en-US/firefox/addon/dark-vcenter/?src=search" target="_blank">Dark-vCenter for Mozilla Firefox</a>

---

The next step is to complete the VIC appliance installation through the establishment of the connection to our vCenter Server as well as Platform Service Controller. Here we have to enter the vCenter Server address (FQDN) and the Single Sign-on credentials for a vSphere administrator account. In my case, I´m using an embedded PSC and thus, I can leave the fields for the External PSC Instance empty.

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180608_105035.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180608_105035.jpg" class="center" width="800"  >}}

If you´ve entered your credentials correctly, you´ll be forwarded to the VIC Getting Started page which you could always open by using the IP-Address or better using the FQDN (of course the short name as well) over port 9443. In my example https://vic01.lab.jarvis.local:9443/

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180608_105155.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180608_105155.jpg" class="center" width="800"  >}}

---
**Continue with:**

<a href="/post/vmware-vsphere-integrated-containers-part-2-vsphere-client-plugin/">**vSphere Integrated Containers Part II: vSphere Client Plug-In**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-3-deployment-of-a-virtual-container-host/">**vSphere Integrated Containers Part III: Deployment of a Virtual Container Host**</a>

<a href="/post/vmware-vsphere-integrated-containers-part-4-docker-run-a-container-vm/">**vSphere Integrated Containers Part IV: docker run a Container-VM**</a>

---

**<center>Thanks for reading!</center>**
