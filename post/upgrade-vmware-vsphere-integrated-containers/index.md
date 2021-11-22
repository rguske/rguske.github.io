# Upgrade vSphere Integrated Containers


<!--more-->

I like upgrades! Because mostly they bring cool new features with them or eliminate disturbing issues.
In this post I´d like to guide you through the upgrade process of <a href="/post/vmware-vsphere-integrated-containers-introduction/">VMware vSphere Integrated Containers </a> from version v1.3.1 to v1.4.1 which is a major upgrade and also to v1.4.3 which is a minor upgrade.

Thanks <a href="https://twitter.com/tom_schwaller" target="_blank">Tom Schwaller</a> for reviewing this article.

First of all and this applies to all upgrades, we have to check the compatibility and dependencys with our infrastructure components. This is for instance our vCenter Server where we have dependencies through the <a href="/post/vmware-vsphere-integrated-containers-part-2-vsphere-client-plugin/">VIC vSphere Client Plug-In</a>!

VIC version 1.4.0 is e.g. compatible with vCenter Server version 6.0.0 U2 until 6.7.0! VIC verison 1.3.0 in contrast to leaves version 6.0.0 U2 as well as 6.5.0. See <a href="https://www.vmware.com/resources/compatibility/sim/interop_matrix.php#interop&2=&149=" target="_blank">VMware Product Interoperability Matrices</a>.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181017_081612.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181017_081612.jpg" class="center" >}}

Of course the same applies to the upgrade path of the solution itself. You can check the <a href="https://www.vmware.com/resources/compatibility/sim/interop_matrix.php#upgrade" target="_blank"> Upgrade Path </a> for every specific VMware solution also over the VMware Product Interoperability Matrices. You´ll find the available upgrade paths for VIC <a href="https://www.vmware.com/resources/compatibility/sim/interop_matrix.php#upgrade&solution=149" target="_blank"> HERE </a>.

Let´s now take a look at the interoperability for vSphere Integrated Containers:

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20180815_084838.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20180815_084838.jpg" class="center" width="800" >}}

We can see that an upgrade from version 1.3.1 to 1.4(.x) is supported and this is for us the final sign to start.

The upgrade itself can be splitted into four phases.

1. VIC Appliance rollout in the desired and supported version
2. Download of the newest VIC Engine Bundle (`vic-machine`)
3. VIC vSphere Client Plug-In upgrade
4. Upgrade of the Virtual Container Host(s)

Before we initiate the deployment of the new VIC Appliance, we´ll validate our used versions. For the VIC Appliance itself we´ll simply start the VMware Remote Console which will then show us the currently used version.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20180807_115237.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20180807_115237.jpg" class="center" width="800" >}}

**Version** (Github versioning): **1.3.1** (<a href="https://git-scm.com/book/en/v2/Git-Basics-Tagging" target="_blank">Tag</a>) **3409** (Build) **132fb13** (<a href="https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository" target="_blank">last Commit</a>)

Next, we´ll check the version of our running Virtual Container Host(s) by using the command `docker info`. 

As described in my previous post <a href="/post/vmware-vsphere-integrated-containers-part-4-docker-run-a-container-vm/"> vSphere Integrated Containers Part IV: docker run a Container-VM </a> we can export our VCH through it´s IP-Address and port via `export DOCKER_HOST=192.168.100.220:2376`.

After we´ve exported it we´ll fire up `docker --tls info` what will show us the following output:

```shell
Containers: 7
 Running: 7
 Paused: 0
 Stopped: 0
Images: 5
Server Version: v1.3.1-16055-afdab46
Storage Driver: vSphere Integrated Containers v1.3.1-16055-afdab46 Backend Engine
VolumeStores: default
vSphere Integrated Containers v1.3.1-16055-afdab46 Backend Engine: RUNNING
 VCH CPU limit: 22800 MHz
 VCH memory limit: 30.49 GiB
 VCH CPU usage: 79 MHz
 VCH memory usage: 1.637 GiB
 VMware Product: VMware vCenter Server
 VMware OS: linux-x64
 VMware OS version: 6.7.0
 Registry Whitelist Mode: disabled.  All registry access allowed.
Plugins:
 Volume: vsphere
 Network: bridge container-net
 Log:
Swarm: inactive
Operating System: linux-x64
OSType: linux-x64
Architecture: x86_64
CPUs: 22800
Total Memory: 30.49GiB
ID: vSphere Integrated Containers
Docker Root Dir:
Debug Mode (client): false
Debug Mode (server): false
Registry: registry.hub.docker.com
Experimental: false
Live Restore Enabled: false
```

The server version which is running within our VCH is v1.3.1-16055-afdab46.

Another way to get the desired info is to make use of the `vic-machine` command line utility with the option `inspect`:

```shell
./vic-machine-darwin inspect \
--target lab-vcsa67-001.lab.jarvis.local/Datacenter-South \
--user adm.jarvis@LAB.JARVIS.LOCAL \
--thumbprint 4F:D3:9B:50:00:31:D9:84:9D:DA:CF:57:21:D6:0D:11:89:78:97:26 \
--name vch-elk
```

The output will also show us the VCH version and the Installer version from which the VCH got deployed as well...

```code
INFO[0000] vSphere password for adm.jarvis@LAB.JARVIS.LOCAL:
INFO[0004] ### Inspecting VCH ####
INFO[0005] Validating target
INFO[0005]
INFO[0005] VCH ID: VirtualMachine:vm-1190
INFO[0005]
INFO[0005] Installer version: v1.3.1-16055-afdab46
INFO[0005] VCH version: v1.3.1-16055-afdab46
INFO[0005]
INFO[0005] VCH upgrade status:
INFO[0005] Installer has same version as VCH
INFO[0005] No upgrade available with this installer version
WARN[0005] Unable to identify address acceptable to host certificate
INFO[0005]
INFO[0005] VCH Admin Portal:
INFO[0005] https://192.168.100.220:2378
INFO[0005]
INFO[0005] Published ports can be reached at:
INFO[0005] 192.168.100.220
INFO[0005]
INFO[0005] Docker environment variables:
INFO[0005] DOCKER_HOST=192.168.100.220:2376
INFO[0005]
INFO[0005] Connect to docker:
INFO[0005] docker -H 192.168.100.220:2376 --tls info
INFO[0005] Completed successfully
```

...and a hint regarding the *VCH Upgrade Status*:

> INFO[0005] VCH upgrade status:
>
> INFO[0005] Installer has same version as VCH
>
> INFO[0005] No upgrade available with this installer version

If we run `vic-machine inspect` executed from a newer VIC-Bundle like 1.4.x for example we´ll see that the installer version differs from the VCH version. But more about this later on.

{{< admonition info "Info" true >}}
By the way! I´ve documented a few examples of `vic-machine` options on my Github-Repo https://github.com/rguske/vic-machine
{{< /admonition >}}

## Phase I

### VIC Appliance Upgrade v1.3.1 to 1.4.1

I suppose that you are familiar with the VIC Appliance deployment because this post described the upgrade of an already deployed VIC appliance but in case not, you can go <a href="/post/vmware-vsphere-integrated-containers-part-1-ova-deployment/" target="_blank"> here</a> first.

**Download** the latest bits through this link: <a href="https://www.vmware.com/go/download-vic" target="_blank">www.vmware.com/go/download-vic</a>.

Start the deployment and be aware of the fact, that _this is NOT an Inplace-Upgrade_ and therefore we have to enter a new IP address as well as FQDN.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20180918_082532.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20180918_082532.jpg" class="center" width="800" >}}

If you wish to reuse the originally IP address after the upgrade, reconfigure the old appliance to use a temporary IP address before you start the upgrade.

VIC Appliance --> Edit Settings --> vApp Options

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181016_112747.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181016_112747.jpg" class="center" width="800" >}}

At this point I´d like to quote the following astracts from the <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/upgrade_appliance.html" target="_blank"> Upgrade section </a>of the official documentation page:

{{< admonition warning "Important" true >}}
**The upgrade process copies data from the old appliance to the new appliance**. Consequently, if you deployed the appliances to a cluster, the virtual disks for the two appliances must be located in the same datastore cluster.

&

**After the new appliance has initialized, do not go to the Getting Started page of the appliance**. Logging in to the Getting Started page for the first time initializes the appliance. Initialization is only applicable to new installations and causes upgraded appliances not to function correctly.

&

You cannot upgrade between untagged, open source builds of the same release like 1.4.3 open source build to 1.4.3 official build for example.
{{< /admonition >}}

After the deployment of the new VIC Appliance we´ll establish a `ssh` connection to it.

```shell
ssh root@192.168.100.161
```

```code
The authenticity of host '192.168.100.161 (192.168.100.161)' can't be established.
ECDSA key fingerprint is SHA256:iVIdb5Sv1pxUsiqelop7DVubJcvjuHHbPSJWBg3No1g.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.100.161' (ECDSA) to the list of known hosts.

##########################################################################
##  SSH access to the vSphere Integrated Containers Appliance can be    ##
##  used in exceptional cases that cannot be handled through standard   ##
##  remote management or CLI tools. This is primarily intended for use  ##
##  in break-fix scenarios, under the guidance of VMware GSS.           ##
##########################################################################

Password:

##########################################################################
##  SSH access to the vSphere Integrated Containers Appliance can be    ##
##  used in exceptional cases that cannot be handled through standard   ##
##  remote management or CLI tools. This is primarily intended for use  ##
##  in break-fix scenarios, under the guidance of VMware GSS.           ##
##########################################################################
```

Use `cd /etc/vmware/upgrade` to go into the directory which contains the `./upgrade.sh` script and execute it. You´ll be promped to enter the neccassary information. As an alternative you can also specify the neccassary arguments upfront like this:

```shell
./upgrade.sh --target 192.168.178.72 \        ### vCenter Server
--username 'administrator@jarvis.local' \     ### User with appropriate permissions
--password '*********'   \                    ### Password
--fingerprint '192.168.178.72 4F:D3:9B...' \  ### vCenter Server Fingerprint
--dc Datacenter-North \                       ### vSphere Datacenter
--embedded-psc \                              ### Platform Service Controller (--embedded-psc or --external-psc)
--appliance-target 192.168.100.160 \          ### VIC Appliance where we´d upgrade from
--appliance-username root \                   ### VIC Appliance root user
--appliance-password VMware1234!?! \          ### root user password
--appliance-version v1.3.1 \                  ### VIC Appliance version where we come from
--destroy                                     ### Destroy the the old appliance after the upgrade is finished.
```

See: <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/upgrade_appliance.html#upgradeoptions" target="_blank"> Specify Command Line Options During Appliance Upgrade </a>

The first time I ran the upgrade it ended up in an error which was related to the "**? letter**" which I´ve used in my password.

```code
root@vic02 [ /etc/vmware/upgrade ]# ./upgrade.sh
-------------------------------
VIC Appliance Upgrade to v1.4.1
-------------------------------

[=] Values containing $ (dollar sign), ` (backquote), \' (single quote), " (double quote), and \ (backslash) will not be substituted properly.
[=] Change any input (passwords) containing these values before running this script.
Enter vCenter Server FQDN or IP: lab-vcsa67-001.lab.jarvis.local
Enter vCenter Administrator Username: administrator@jarvis.local
Enter vCenter Administrator Password:
If using an external PSC, enter the FQDN of the PSC instance (leave blank otherwise):
If using an external PSC, enter the PSC Admin Domain (leave blank otherwise):
govc: dial tcp: address tcp/VMware1234!: unknown port
[=]
[=] -------------------------
[=] Upgrade failed.
[=] Please save /var/log/vmware/upgrade.log and contact VMware support.
[=] -------------------------
[=]
```

The output `govc: dial tcp: address tcp/VMware1234!: unknown port` gives us first indications that it might be closely connected with the letters in my password. There are two ways to bypass this.

1. **[The easy way]** Use another user with the needed permissions (see <a href="https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.security.doc/GUID-21EF0E93-0003-4D41-AA11-A73824E1A88C.html" target="_blank"> *vSphere Extension Privileges*)</a> and without including an **?** in the password.
2. **[The <a href="https://github.com/vmware/govmomi" target="_blank"> govmomi </a> way]** Use the workaround described here: <a href="https://github.com/vmware/govmomi/issues/1203" target="_blank"> github.com/vmware/govmomi/issues/1203 </a>

I´ve chosen the easy way ;-) and this is how it looks when the upgrade to **v1.4.1** went well:

```code
root@vic02 [ /etc/vmware/upgrade ]# ./upgrade.sh
-------------------------------
VIC Appliance Upgrade to v1.4.1
-------------------------------

2018-08-16 21:20:30 [=] Values containing $ (dollar sign), ` (backquote), \' (single quote), " (double quote), and \ (backslash) will not besubstituted properly.
2018-08-16 21:20:30 [=] Change any input (passwords) containing these values before running this script.
Enter vCenter Server FQDN or IP: 192.168.178.72
Enter vCenter Administrator Username: admin@jarvis.local
Enter vCenter Administrator Password:
If using an external PSC, enter the FQDN of the PSC instance (leave blank otherwise):
If using an external PSC, enter the PSC Admin Domain (leave blank otherwise):

Please verify the vCenter IP and TLS fingerprint: 192.168.178.72 4F:D3:9B:50:00:31:D9:84:9D:DA:CF:57:21:D6:0D:11:89:78:97:26
Is the fingerprint correct? (y/n): y
Enter vCenter Datacenter of the old VIC appliance: Datacenter-North
Enter old VIC appliance IP: 192.168.100.160
Enter old VIC appliance username: root
2018-08-16 21:21:25 [=]
-------------------------
Starting upgrade 2018-08-16 21:20:30 +0000 UTC

2018-08-16 21:21:26 [=] Please enter the VIC appliance password for root@192.168.100.160
Password:
2018-08-16 21:21:36 [=]
2018-08-16 21:21:36 [=] Detected old appliance's version as v1.3.1.
2018-08-16 21:21:36 [=] If the old appliance's version is not detected correctly, please enter "n" to abort the upgrade and contact VMware support.
2018-08-16 21:21:36 [=]
2018-08-16 21:21:36 [=] Do you wish to proceed with upgrade? [y/n]
y
2018-08-16 21:21:43 [=] Continuing with upgrade
2018-08-16 21:21:43 [=]
2018-08-16 21:21:45 [=] Waiting for old VIC appliance to power off
2018-08-16 21:22:00 [=] Waiting for old VIC appliance to power off
2018-08-16 21:22:18 [=] Migrating old disks to new VIC appliance...
2018-08-16 21:22:19 [=] Copying old data disk. Please wait.
2018-08-16 21:22:25 [=] Copying old database disk. Please wait.
2018-08-16 21:22:32 [=] Copying old log disk. Please wait.
2018-08-16 21:22:41 [=] Finished attaching migrated disks to new VIC appliance
2018-08-16 21:22:41 [=] Preparing upgrade environment
2018-08-16 21:22:41 [=] Disabling and stopping Admiral and Harbor
2018-08-16 21:22:41 [=] Waiting for register appliance...
2018-08-16 21:22:51 [=] Waiting for register appliance...
2018-08-16 21:23:01 [=] Waiting for register appliance...
2018-08-16 21:23:40 [=] Finished preparing upgrade environment
2018-08-16 21:23:40 [=]
-------------------------
Starting Admiral Upgrade 2018-08-16 21:20:30 +0000 UTC

2018-08-16 21:23:40 [=] Performing pre-upgrade checks
2018-08-16 21:23:40 [=] Starting Admiral upgrade
2018-08-16 21:24:28 [=] Updating Admiral configuration
2018-08-16 21:24:28 [=] Restarting Admiral
2018-08-16 21:25:17 [=]
-------------------------
Starting Harbor Upgrade 2018-08-16 21:20:30 +0000 UTC

2018-08-16 21:25:17 [=] Performing pre-upgrade checks
2018-08-16 21:25:17 [=] Starting Harbor upgrade
2018-08-16 21:25:17 [=] [=] Shutting down Harbor
2018-08-16 21:25:17 [=] [=] Migrating Harbor configuration and data
2018-08-16 21:25:17 [=] Testing database credentials...
2018-08-16 21:25:18 [=] Backing up harbor config...
2018-08-16 21:25:23 [=] [=] Successfully migrated Harbor configuration and data
2018-08-16 21:25:23 [=] Harbor upgrade complete
2018-08-16 21:25:23 [=] Starting Harbor
2018-08-16 21:25:33 [=] Enabling and starting Admiral and Harbor
2018-08-16 21:25:33 [=]
2018-08-16 21:25:33 [=] -------------------------
2018-08-16 21:25:33 [=] Upgrade completed successfully. Exiting.
2018-08-16 21:25:33 [=] -------------------------
```

All neccessary information about the upgrade are also available in the **upgrade.log** which is deposited under `/var/log/vmware` on the VIC appliance.

## Phase II

### Download of the newest VIC Engine Bundle (`vic-machine`)

In phase two we´ll download the VIC Engine Bundle which includes the following:

1. Scripts for the *VIC vSphere Client Plug-In* like install, upgrade, or remove
2. The latest version of the `vic-machine` utility

I´ve already covered this topic in my previous post <a href="/post/vmware-vsphere-integrated-containers-part-3-deployment-of-a-virtual-container-host/" target="_blank"> vSphere Integrated Containers Part III: Deployment of a Virtual Container Host</a> right at the beginning.

Open the VIC *Getting Started* page by entering your VIC Address with port 9443 into a browser and find the download-button under the point *Infrastructure Deployment Tools*.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181001_113850.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181001_113850.jpg" class="center" width="800" >}}

Otherwise you can go directly to: https://*your VIC Appliance address*:9443/files/

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181001_113551.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181001_113551.jpg" class="center" width="500" >}}

## Phase III

### VIC vSphere Client Plug-In upgrade

Now that we have the new VIC Enginge Bundle available we´ll make use of the *Upgrade Script* for the *VIC vSphere Client Plug-In*.

{{< admonition note "Note" true >}}
This applies only until version VIC 1.4.1!
{{< /admonition >}}

Use a terminal and you´ll find the script deposited in the path **/vic141/ui/vcsa** on your local computer or remote execution host (wherever you´ve downloaded it).

```shell
total 72
drwxr-xr-x@ 6 rguske  staff    192 Aug 16 23:22 .
drwxr-xr-x@ 8 rguske  staff    256 Jul  3 17:37 ..
-rw-r--r--@ 1 rguske  staff    937 Aug 16 23:22 configs
-rwxr-xr-x@ 1 rguske  staff  10364 Jun 22 22:28 install.sh
-rwxr-xr-x@ 1 rguske  staff   6002 Jun 22 22:28 uninstall.sh
-rwxr-xr-x@ 1 rguske  staff  12059 Jun 22 22:28 upgrade.sh
```

Execute `./upgrade.sh` so that the script starts prompting you after neccessary information regarding the vCenter Server target and user credentials. This is how it will look like:

```code
-------------------------------------------------------------
This script will upgrade vSphere Integrated Containers plugin
for vSphere Client (HTML) and vSphere Web Client (Flex).

Please provide connection information to the vCenter Server.
-------------------------------------------------------------
Enter FQDN or IP to target vCenter Server: lab-vcsa67-001.lab.jarvis.local
Enter your vCenter Administrator Username: administrator@jarvis.local
Enter your vCenter Administrator Password:

SHA-1 key fingerprint of host 'lab-vcsa67-001.lab.jarvis.local' is '4F:D3:9B:50:00:31:D9:84:9D:DA:CF:57:21:D6:0D:11:89:78:97:26'
Are you sure you trust the authenticity of this host (yes/no)? yes

-------------------------------------------------------------
Checking existing plugins...
-------------------------------------------------------------
Plugin with key 'com.vmware.vic' is already registered with VC. (Version: 1.3.1.687)
Plugin with key 'com.vmware.vic.ui' is already registered with VC (Version: 1.3.1.687)
The version you are about to install is '1.4.1.1262'.

Are you sure you want to continue (yes/no)? yes

-------------------------------------------------------------
Preparing to upgrade vCenter Extension vSphere Integrated Containers-FlexClient...
-------------------------------------------------------------
Aug 29 2018 23:12:07.794+02:00 INFO  ### Installing UI Plugin ####
Aug 29 2018 23:12:07.919+02:00 INFO  Removing existing plugin to force install
Aug 29 2018 23:12:08.164+02:00 INFO  Removed existing plugin
Aug 29 2018 23:12:08.164+02:00 INFO  Installing plugin
Aug 29 2018 23:12:08.484+02:00 INFO  Installed UI plugin

-------------------------------------------------------------
Preparing to upgrade vCenter Extension vSphere Integrated Containers-H5Client...
-------------------------------------------------------------
Aug 29 2018 23:12:08.535+02:00 INFO  ### Installing UI Plugin ####
Aug 29 2018 23:12:08.664+02:00 INFO  Removing existing plugin to force install
Aug 29 2018 23:12:08.730+02:00 INFO  Removed existing plugin
Aug 29 2018 23:12:08.730+02:00 INFO  Installing plugin
Aug 29 2018 23:12:08.865+02:00 INFO  Installed UI plugin
Aug 29 2018 23:12:09.230+02:00 WARN  ignoring potential product VM vm-321: not powered on
Aug 29 2018 23:12:09.237+02:00 INFO  Found 1 VM(s) tagged as OVA
Aug 29 2018 23:12:09.247+02:00 INFO  Attempting to configure ManagedByInfo
Aug 29 2018 23:12:09.485+02:00 INFO  Successfully configured ManagedByInfo
```

If all the entered information were correct and most of the time it depends on the correct user privileges, it will output...

```shell
--------------------------------------------------------------
Upgrade successful. Restart the vSphere Client services. All vSphere Client users must log out and log back in again to see the vSphere Integrated Containersplug-in.
Exited successfully
```

Finally and as requested in the last lines of the above output, restart the vSphere Client services on your vCenter Server Appliance.

```shell
service-control --stop vsphere-ui && service-control --start vsphere-ui
service-control --stop vsphere-client && service-control --start vsphere-client
```

{{< image src="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100412.jpg" src-s="/img/posts/201807_vic_getting_started/CapturFiles-20180616_100412.jpg" class="center" width="800" >}}

In case you are missing something here in my description, you´ll perhaps find your answer here: <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/upgrade_h5_plugin_vcsa.html" target="_blank"> Manually Upgrade the vSphere Client Plug-In on vCenter Server Appliance </a>

## Upgrading v1.4.1 to v1.4.3

This phase (II) of the VIC Upgrade-process is now obsolete with version 1.4.3. The upgrade of the vSphere (H5) Plug-In is now integrated into the whole process. As you can see through the following output, the question after the *VIC UI Plugin* upgrade appeared when you start the VIC Appliance upgrade:

`Upgrade VIC UI Plugin? (y/n):` and of course I answered it with `y` (=yes)...what else?!

```code
root@vic01 [ /etc/vmware/upgrade ]# ./upgrade.sh
-------------------------------
VIC Appliance Upgrade to v1.4.3
-------------------------------

2018-10-09 21:43:15 [=] Values containing $ (dollar sign), ` (backquote), \' (single quote), " (double quote), and \ (backslash) will not be substituted properly.
2018-10-09 21:43:15 [=] Change any input (passwords) containing these values before running this script.
Enter vCenter Server FQDN or IP: lab-vcsa67-001.lab.jarvis.local
Enter vCenter Administrator Username: admin@jarvis.local
Enter vCenter Administrator Password:
If using an external PSC, enter the FQDN of the PSC instance (leave blank otherwise):
If using an external PSC, enter the PSC Admin Domain (leave blank otherwise):

Please verify the vCenter IP and TLS fingerprint: lab-vcsa67-001.lab.jarvis.local 4F:D3:9B:50:00:31:D9:84:9D:DA:CF:57:21:D6:0D:11:89:78:97:26
Is the fingerprint correct? (y/n): y
Enter vCenter Datacenter of the old VIC appliance: Datacenter-North
Enter old VIC appliance IP: 192.168.100.161
Enter old VIC appliance username: root
Upgrade VIC UI Plugin? (y/n): y
2018-10-09 21:43:58 [=]
-------------------------
Starting upgrade 2018-10-09 21:43:15 +0000 UTC

2018-10-09 21:44:00 [=] Please enter the VIC appliance password for root@192.168.100.161
The authenticity of host '192.168.100.161 (192.168.100.161)' can't be established.
ECDSA key fingerprint is SHA256:MTj3Ak65Uem4uiipiygyOpVJuik7iawz09hFsz7tSM4.
Are you sure you want to continue connecting (yes/no)? yes
Password:
2018-10-09 21:44:09 [=]
2018-10-09 21:44:09 [=] Detected old appliance's version as v1.4.1.
2018-10-09 21:44:09 [=] If the old appliance's version is not detected correctly, please enter "n" to abort the upgrade and contact VMware support.
2018-10-09 21:44:09 [=]
2018-10-09 21:44:09 [=] Do you wish to proceed with upgrade? [y/n]
y
2018-10-09 21:44:17 [=] Continuing with upgrade
2018-10-09 21:44:17 [=]
2018-10-09 21:44:19 [=] Waiting for old VIC appliance to power off
2018-10-09 21:44:35 [=] Waiting for old VIC appliance to power off
2018-10-09 21:44:50 [=] Waiting for old VIC appliance to power off
2018-10-09 21:45:05 [=] Waiting for old VIC appliance to power off
2018-10-09 21:45:20 [=] Waiting for old VIC appliance to power off
2018-10-09 21:45:39 [=] Migrating old disks to new VIC appliance...
2018-10-09 21:45:40 [=] Copying old data disk. Please wait.
2018-10-09 21:45:45 [=] Copying old database disk. Please wait.
2018-10-09 21:45:51 [=] Copying old log disk. Please wait.
2018-10-09 21:45:59 [=] Finished attaching migrated disks to new VIC appliance
2018-10-09 21:45:59 [=] Preparing upgrade environment
2018-10-09 21:45:59 [=] Disabling and stopping Admiral and Harbor
2018-10-09 21:45:59 [=] Registering the appliance in PSC
2018-10-09 21:46:00 [=] Waiting for register appliance...
2018-10-09 21:46:39 [=] Finished preparing upgrade environment
2018-10-09 21:46:39 [=]
-------------------------
Starting VIC UI Plugin Upgrade 2018-10-09 21:43:15 +0000 UTC

2018-10-09 21:46:41 [=]
-------------------------
Starting Admiral Upgrade 2018-10-09 21:43:15 +0000 UTC

2018-10-09 21:46:41 [=] Performing pre-upgrade checks
2018-10-09 21:46:41 [=] Starting Admiral upgrade
2018-10-09 21:47:22 [=] Updating Admiral configuration
2018-10-09 21:47:23 [=] Restarting Admiral
2018-10-09 21:48:12 [=]
-------------------------
Starting Harbor Upgrade 2018-10-09 21:43:15 +0000 UTC

2018-10-09 21:48:12 [=] Performing pre-upgrade checks
2018-10-09 21:48:12 [=] Starting Harbor upgrade
2018-10-09 21:48:12 [=] [=] Shutting down Harbor
2018-10-09 21:48:12 [=] [=] Migrating Harbor configuration and data
2018-10-09 21:48:12 [=] Testing database credentials...
2018-10-09 21:48:14 [=] Backing up harbor config...
2018-10-09 21:48:19 [=] [=] Successfully migrated Harbor configuration and data
2018-10-09 21:48:19 [=] Harbor upgrade complete
2018-10-09 21:48:19 [=] Starting Harbor
2018-10-09 21:48:30 [=] Enabling and starting Admiral and Harbor
2018-10-09 21:48:30 [=]
2018-10-09 21:48:30 [=] -------------------------
2018-10-09 21:48:30 [=] Upgrade completed successfully. Exiting. All vSphere Client users must log out and log back in again twice to see the vSphere Integrated Containers plug-in.
2018-10-09 21:48:30 [=] -------------------------
2018-10-09 21:48:30 [=]
```

In the vSphere Client you´ll see an information that plug-ins have been updated and they´ll be ready for use next time you log into vSphere Client.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181009_115056.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181009_115056.jpg" class="center" width="800" >}}

Re-Login

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181009_115536.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181009_115536.jpg" class="center" width="800" >}}

## Phase IV

### Upgrade of the Virtual Container Host(s)

Last but not least we´ll upgrade our Virtual Container Host(s). This will also be done by using the `vic-machine` utility with the option `upgrade`. Before I begin, I´ll check all current versions of my running VCH´s.

Use `docker --tls version` to get the desired information.

For my VCH based and deployed on version 1.3.1 the output looks like:

```shell
Client:
 Version:           18.06.1-ce
 API version:       1.25 (downgraded from 1.38)
 Go version:        go1.10.3
 Git commit:        e68fc7a
 Built:             Tue Aug 21 17:21:31 2018
 OS/Arch:           darwin/amd64
 Experimental:      false

Server:
 Engine:
  Version:          1.3.1
  API version:      1.25 (minimum version 1.19)
  Go version:       go1.8.3
  Git commit:       afdab46
  Built:            Mon Feb  5 15:35:58 2018
  OS/Arch:          linux/amd64
  Experimental:     true
```

...and for my VCH based on version 1.4.1 it looks like this:

```shell
Client:
 Version:           18.06.1-ce
 API version:       1.25 (downgraded from 1.38)
 Go version:        go1.10.3
 Git commit:        e68fc7a
 Built:             Tue Aug 21 17:21:31 2018
 OS/Arch:           darwin/amd64
 Experimental:      false

Server:
 Engine:
  Version:          1.4.1
  API version:      1.25 (minimum version 1.19)
  Go version:       go1.8.7
  Git commit:       b3a6192
  Built:            Tue Jul  3 15:05:57 2018
  OS/Arch:          linux/amd64
  Experimental:     true
```

Now let´s upgrade our VCH from version 1.4.1 to our latest version:

<center> {{< tweet user="vmw_rguske" id="1041955154910740480" >}} </center>

The `upgrade` option from `vic-machine` requires the VM-ID from our VCH and we can get this ID by using `vic-machine inspect`. 

That´s the point were you´ll hopefully say to yourself "*AH! We´ve already made use of this option to get the VCH version displayed*" and you`re right "*Watson*" ;-)!

And I´ve also mentioned that I´ll come back to this point. Because we now will fire up this command from the new VIC Engine Bundle...

```shell
./vic-machine-darwin inspect \
--target lab-vcsa67-001.lab.jarvis.local/Datacenter-North \
--user adm.jarvis@LAB.JARVIS.LOCAL \
--thumbprint 4F:D3:9B:50:00:31:D9:84:9D:DA:CF:57:21:D6:0D:11:89:78:97:26 \
--name vch-elk
```

...and the output will show us this time the following:

```code
INFO[0000] vSphere password for adm.jarvis@LAB.JARVIS.LOCAL:
INFO[0004] ### Inspecting VCH ####
INFO[0004] Validating target
INFO[0005]
INFO[0005] Installer version: v1.4.3-20031-fa9732d
INFO[0005] VCH version: v1.4.1-19562-b3a6192
INFO[0005]
INFO[0005] VCH upgrade status:
INFO[0005] Upgrade available
WARN[0005] Unable to identify address acceptable to host certificate, using assigned client IP as host address.
INFO[0005]
INFO[0005] VCH ID: vm-1190
INFO[0005]
INFO[0005] VCH Admin Portal:
INFO[0005] https://192.168.100.220:2378
INFO[0005]
INFO[0005] Published ports can be reached at:
INFO[0005] 192.168.100.22
INFO[0005]
INFO[0005] Management traffic will use:
INFO[0005] 192.168.100.220
INFO[0005]
INFO[0005] Docker environment variables:
INFO[0005] DOCKER_HOST=192.168.100.220:2376
INFO[0005]
INFO[0005] Connect to docker:
INFO[0005] docker -H 192.168.100.220:2376 --tls info
INFO[0005] Completed successfully
```

> INFO[0005] **Installer version: v1.4.3-20031-fa9732d**
>
> INFO[0005] **VCH version: v1.4.1-19562-b3a6192**
>
> INFO[0005] VCH upgrade status:
>
> INFO[0005] **Upgrade available**

*Upgrade available* sounds good to me! Here we go:

```shell
./vic-machine-darwin upgrade \
--target lab-vcsa67-001.lab.jarvis.local/Datacenter-North \
--user administrator@jarvis.local \
--id vm-1190 \
--force
```

...and a couple of seconds later...

```code
INFO[0000] vSphere password for administrator@jarvis.local:
INFO[0004] ### Upgrading VCH ####
INFO[0004] Validating target
INFO[0004]
INFO[0004] VCH ID: VirtualMachine:vm-1190
INFO[0005] Creating directory [lab-ds-001] vch-elk
INFO[0005] Datastore path is [lab-ds-001] vch-elk
INFO[0005] Uploading ISO images
INFO[0006] Uploading appliance.iso as V1.4.3-20031-FA9732D-appliance.iso
INFO[0016] Uploading bootstrap.iso as V1.4.3-20031-FA9732D-bootstrap.iso
INFO[0023] Switching appliance iso to [lab-ds-001] vch-elk/V1.4.3-20031-FA9732D-appliance.iso
INFO[0023] Setting VM configuration
WARN[0024] If ops-user (jarvis@lab) belongs to the Administrators group, permissions on some resources might have been restricted
INFO[0025] Waiting for IP information
INFO[0025] Waiting for major appliance components to launch
INFO[0035] Obtained IP address for client interface: "192.168.100.220"
INFO[0035] Checking VCH connectivity with vSphere target
INFO[0035] vSphere API Test: https://lab-vcsa67-001.lab.jarvis.local vSphere API target responds as expected
INFO[0121] Completed successfully
```

Version check (`docker --tls version`) again!

```shell
Client:
 Version:           18.06.1-ce
 API version:       1.25 (downgraded from 1.38)
 Go version:        go1.10.3
 Git commit:        e68fc7a
 Built:             Tue Aug 21 17:21:31 2018
 OS/Arch:           darwin/amd64
 Experimental:      false

Server:
 Engine:
  Version:          1.4.3
  API version:      1.25 (minimum version 1.19)
  Go version:       go1.8.7
  Git commit:       fa9732d
  Built:            Fri Sep 14 08:14:14 2018
  OS/Arch:          linux/amd64
  Experimental:     true
```

## Conclusion

We´ve went through each individual phase of the upgrade-process of vSphere Integrated Containers. The upgrade to a newer version is simple and improvements were made in version 1.4.3.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_113631.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_113631.jpg" class="center" width="800" >}}

When upgrading to a newer version we just have to observe the following aspects:

- Upgrading vSphere Integrated Containers is not that difficult and it´s well documented. <a href="https://vmware.github.io/vic-product/assets/files/html/1.4/vic_vsphere_admin/upgrading_vic.html" target="_blank"> *Upgrading vSphere Integrated Containers*</a>
- Upgrade between untagged, open source builds of the same release like 1.4.3 open source build to 1.4.3 official build is not supported
- The upgrade of the vSphere Client Plug-In is now in the VIC Appliance upgrade integrated with version 1.4.3
- While upgrading the Virtual Conatiner Host(s), the VCH get´s rebooted which means that when you have NOT configured a dedicated Container-Network for your VCH, your application cannot be reached over the VCH IP-Address and it´s exposed port.

{{< admonition tip "Tip" true >}}
*To avoid this, make use of one of the big benefits vSphere Integrated Containers can leverage here and configure a* ***dedicated Container-Network to your VCH!***

*In addition have a look at <a href="/post/vmware-vsphere-integrated-containers-part-4-docker-run-a-container-vm/">vSphere Integrated Containers Part IV: docker run a Container-VM</a>*
{{< /admonition >}}


You can easily test this by instantiating a (vmwarecna) <a href="https://hub.docker.com/r/vmwarecna/nginx/" target="_blank"> NGINX</a> Container-VM with the `--net` option, start upgrading the Virtual Container Host and see if the C-VM will become unreachable.

Example: `docker --tls run --name vmwarenginx --net vch-blog-net -d -p 8080:80 vmwarecna/nginx`

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_104814.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_104814.jpg" class="center" width="800" >}}

While upgrading the VCH, the NGINX Container-VM is still reachable.

{{< image src="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_105153.jpg" src-s="/img/posts/201808_post_vic_upgrade/CapturFiles-20181014_105153.jpg" class="center" width="800" >}}

Please find the Release Notes for vSphere Integrated Containers here: <a href="https://docs.vmware.com/en/VMware-vSphere-Integrated-Containers/" target="_blank">VMware vSphere Integrated Containers Docs</a>

I hope you won´t find this post not too long for this kind of topic, but I wanted to be as detailed as possible.

