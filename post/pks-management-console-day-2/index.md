# VMware Enterprise PKS Management Console (EPMC) - Day 2


<!--more-->

In the <a href="https://rguske.github.io/post/pks-management-console-day-1/" target="_blank">previous blog post</a> we went through the Day-1 capabilities of the new version of the Enterprise PKS Management Console (EPMC). Let´s have a look at what´s in for **Day-2**.

## Summary Page

The summary page of your Enterprise PKS deployment will appear after clicking the **Enterprise PKS** button at the top left corner. This page will give you an detailed overview of all the PKS Management Plane components including important data like, *Service State*, *Version Numbers*, *IP addresses* and also *Hyperlinks* either to the relevant Virtual Machine in vSphere or to the Website of an Management-Component like, NSX-T or the Pivotal Ops Manager.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191116_062030.jpg" caption="Figure I: Enterprise PKS deployment summary" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191116_062030.jpg" class="center" >}}

## Identity Management

Do you remember the following?

```shell
uaac user add jarvis --emails jarvis@jarvis.lab -p Password0815
```

With EPMC you can simply add users and groups through the **Identity Management** tab. Click on *Add Users* and fill out the required fields. In this version of EPMC, you cannot add individual LDAP or SAML users. Instead of, you have to use the *Groups* tab for LDAP or SAML authentication.

{{< admonition Note "Note" true >}}
This release of Enterprise PKS Management Console does not support assigning roles to individual LDAP or SAML users. To assign roles to LDAP or SAML users, use user groups.

Source: https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-console-console-identity-management.html
{{< /admonition >}}

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191124_052517.jpg" caption="Figure II: Identity Management" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191124_052517.jpg" class="center" >}}

There are two roles which can be assigned. The **Cluster Manager**, to <a href="https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-control-plane.html" target="_blank">manage the lifecycle</a> of the individual Kubernetes Clusters and the **PKS Administrator**, which manage the Enterprise PKS infrastructure.

Depending on what you have configured in Step 3 (Identity) during your deployment configuration, you can enter the distinguished name of an existing LDAP group (`cn=,ou=,dc=`) or the name of your SAML identity provider group.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_091051.jpg" caption="Figure III: Identity Management" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_091051.jpg" class="center" >}}

## Deployment Metadata

This section is kind of an *"Documentation Page"* for your Enterprise PKS deployment. You will find all the important Metadata beginning with *IP-addresses, FQDNs, credentials (username & password), certificates* up to the *Enterprise PKS UAA Encryption Passphrase*.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_073824.jpg" caption="Figure IV: Deployment Metadata" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_073824.jpg" class="center" >}}

**Just as an example what´s also in this section.**

You can use the EPMC as a Jumphost for making use of the BOSH CLI to operationalise PKS. The necessary environment variables, which have to be exported, can be found in the *Metadata* section under the entry **BOSH CLI invocation from console appliance**, as you can see on *Figure IV*.

Once you have exported the environment variables, you can issue commands like:

```shell
bosh vms
```

to get the state of your deployment or

```shell
bosh tasks --recent=20
```

to see the last 20 recent tasks of BOSH.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191124_062351.jpg" caption="Figure V: ssh´d into EPMC | export env variables (Metadata Section) | show deployment state" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191124_062351.jpg" class="center" >}}

## Kubernetes cluster creation

To create a new Kubernetes cluster in Enterprise PKS, you have to make use of the <a href="https://network.pivotal.io/products/pivotal-container-service" target="_blank">**PKS CLI**</a>. Login into the PKS API Endpoint with a previously created user, which is in my example the *svcpksmgr* to which I have assigned the *Cluster Manager* role, and execute the following:

```shell
pks login -a pks.jarvis.lab -u svcpksmgr -p VMware1! --ca-cert pks.crt*
```

{{< admonition tip "Tip" true >}}
The PKS certificate can be found via the *Metadata* tab.
{{< /admonition >}}

After a successful login, create a new Kubernetes cluster in the desired "T-Shirt size" (`--plan`).

```shell
pks create-cluster k8s-small --external-hostname k8s-small.jarvis.lab --plan small
```

You can use

```shell
watch -n 2 pks cluster k8s-small
```
to watch the creation-progress of your new K8s-cluster.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_091448.jpg" caption="Figure VI: Kubernetes cluster deployment" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_091448.jpg" class="center" >}}

## Kubernetes cluster information

Again! The **Enterprise PKS Management Console** brings tons of benefits to operators, by beeing the single pane of glass for your Enterprise PKS/ your Kubernetes environment. The following screenshot gives you an idea what I mean.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_100104.jpg" caption="Figure VII: Kubernetes cluster Overview" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_100104.jpg" class="center" >}}

You can see, that besides the creation date of your cluster, the version of it, the count of Master, Worker and Namespaces OR Storage related information, it also shows you all relevant details regarding Networking like: the assigned <a href="https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-network-profiles-define.html" target="_blank">Network Profile</a>, the connected T0 and T1-Router, the Logical Switch, Load Balancer information as well as Node and POD networking data.

By selecting *Open Kubernetes Dashboard* or *Access Cluster*, a new window appears in both cases with the necessary command-lines in it, which you can either use to get access to the K8s-Dashboard or the Cluster. Otherwise, you can pass it on to the colleague who is eagerly waiting for it.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_043048.jpg" caption="Figure VIII: Access Kubernetes Cluster and Dashboard" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_043048.jpg" class="center" >}}

If you have multiple Kubernetes clusters across multiple Availability Zones deployed, you will get the right synopsis on the **Nodes** tab.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_095912.jpg" caption="Figure IX: Kubernetes Master & Worker" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_095912.jpg" class="center" >}}

Having an understanding of how Enterprise PKS works with NSX-T is crucial. I don´t want to stress this out here, but as a reminder:

{{< admonition note "Note" true >}}
Each Logical Switch will have a Tier1 Logical Router as a Default Gateway. For the Tier1-Cluster Router, the first available subnet from Nodes-IP-Block will be used with the first IP address as the Default Gateway.

For all Kubernetes Name Spaces, a subnet will be derived from the Pods-IP-Block
A SNAT entry will be configured in the Tier0 Logical Router for each Name Space using the IP-Pool-VIPs since it is the Routable one in the enterprise network.
{{< /admonition >}}

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191117_091820.jpg" caption="Figure X: Logical diagram for created Logical Routers and Switches" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191117_091820.jpg" class="center" >}}

*Source: <https://blogs.vmware.com/networkvirtualization/2019/06/kubernetes-and-vmware-enterprise-pks-networking-security-operations-with-nsx-t-data-center.html/>*

The **Namespace** tab provides you the Hyperlinks to the NSX-T *Logical Switches (POD Network)* as well as the *T1-Router* were all the Kubernetes Namespaces are connected to.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_100200.jpg" caption="Figure XI: Kubernetes Namespaces" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191113_100200.jpg" class="center" >}}

## Upgrade Enterprise PKS + Kubernetes

Upgrading your VMware Enterprise PKS Management Plane components as well as your Kubernetes cluster(s) can also be done via the EPMC. If a new version of PKS is available, you´ll be notified by a message at the top. But before you can run the upgrade, you have to migrate your deployment to the newer version of the EPMC first. Therefore:

- Download the new version via https://downloads.vmware.com
- Deploy the OVA
- If you deployed your old EPMC with a static IP and you would like to re-use it for the new version, just *Shutdown* the old appliance, *reconfigure* the vApp with a temporary IP address and *Power on* the appliance again (this should be done before rolling out the new one :wink:)
- Login in into the new EPMC and select this time **Upgrade**. See <a href="https://rguske.github.io/post/pks-management-console-day-1/" target="_blank">*Figure IV: Install or Upgrade Option*</a>

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_022912.jpg" caption="Figure XII: Upgrade Enterprise PKS Management Console" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_022912.jpg" class="center" >}}

**BTW:** As you can see on the screenshot above and below, I´ve duplicated the tab of my browser to highlight a pretty neat feature of the EPMC. By clicking the **?** icon, which can be found at the top bar as well as in some of the sections, a Contextual help will appear.

## VMware Enterprise PKS Component Patches

Unfortunately, this section currently applies to the Enterprise PKS Management Console only. It´s planned to have the ability to patch the PKS Management Plane components in a future release too. For now applies, if an patch for the EPMC is available, you will be notified via a message at the top bar:

`There are new patches to the VMware Enterprise PKS components`.

{{< image src="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_022956.jpg" caption="Figure XIII: VMware Enterprise PKS Component Patches" src-s="/img/posts/201911_pksmgmtconsole/CapturFiles-20191123_022956.jpg" class="center" >}}

---

**<center>Thanks for reading.</center>**

---

## Resources

   **Site**                                       |   **URL**
:-------------------------------------------------:|-------------------------------------------------------------:
**VMware Docs EPMC**               | https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-console-console-index.html
**Pivotal Docs EPMC**             | http://docs-pcf-staging.cfapps.io/pks/1-6/console/deploy-console-ova.html
**VMware Release Notes**           | https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/rn/VMware-Enterprise-PKS-16-Release-Notes.html
**VMware VVD 5.1 - PKS**     | https://docs.vmware.com/en/VMware-Validated-Design/5.1/sddc-architecture-and-design-for-vmware-enterprise-pks-with-vmware-nsx-t-workload-domains/
**EPMC YAML Configuration import** | https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-console-deploy-ent-pks-yaml.html
**VMware Docs - PKS Control Plane**              | https://docs.vmware.com/en/VMware-Enterprise-PKS/1.6/vmware-enterprise-pks-16/GUID-control-plane.html
**Download via VMware**               | https://my.vmware.com/web/vmware/info/slug/infrastructure_operations_management/vmware_enterprise_pks/1_6
**Download via Pivotal**               | https://network.pivotal.io/products/pivotal-container-service
**Download Pivotal CLI**               | https://network.pivotal.io/products/pivotal-container-service
