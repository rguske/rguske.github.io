# VMware Event Broker Appliance - vSphere HA Event Notification Function


## Community

I recently had the pleasure to suppport a member out of the [VMware Event Broker Appliance (VEBA) Slack Community](https://vmwarecode.slack.com/archives/CQLT9B5AA) with his function example contribution to the [VEBA](https://vmweventbroker.io/) project. His name is Bob. He introduced his Powershell/ PowerCLI function on a remote session to me and I was immediately thrilled about what the function does. It covers a use case that one of my customers brought up some time ago when I first introduced VEBA to them.

Bob's engagement was really contagious and getting work done was quite fun. **All credits goes out to Bob!**

## Code

Bob developed a script some time ago, which functionality basically is, to send out a notification via Email after a vSphere HA event occurred. This Email will have the affected host mentioned as well as all affected VMs which has been restarted through vSphere HA. The problem was, that the execution of the script was a manual task every time a ESXi host outage took place...**at least until he became aware of VEBA**.

{{< admonition quote "Quote" true >}}
*"...I knew VEBA would be a prime place to attempt to do this for me."*
{{< /admonition >}}

Then he started digging into it.

There is really good material available on *"Getting started with VEBA"* like for example this [Series](http://www.patrickkremer.com/veba/) by [Patrick Kremer](https://twitter.com/KremerPatrick) as well as articles and documentation on *"Writing your own function"* like e.g. [Writing your first Serverless Function](https://medium.com/@pkblah/writing-your-first-serverless-function-23508cb4ea11) by [Partheeban Kandasamy (PK)](https://twitter.com/pkblah) and also here at the [Documentation](https://vmweventbroker.io/kb/contribute-functions) section at VEBA's homepage.

**You don't have to reinvent the wheel!** If you have great `code`, like Bob, take it, make your hands dirty and write your first function in any of your preferred programming language. It is not that complicated and I promise you the joy of success when your first function does exactly what you built it for.

If you don't have own code available, start looking here: [VMware {code} Sample Exchange](https://code.vmware.com/samples?categories=Sample&keywords=&tags=PowerShell%7CVMware%20PowerCLI&groups=&filters=&sort=dateDesc&page=). The community is diligent and willing to share.

## Contribution

You are probably wondering when I finally introduce the function, right? The reason for my long introduction in this post is that I want to encourage you to start building your own function(s) to unlock the potential of event-driven automation in your data center.

Like Bob successfully did. :grin:

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200629_040718.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200629_040718.jpg" class="center" >}}

Here we go and waiting for it to get merged: **VMware Event Broker Appliance** <i class='fab fa-github fa-fw'></i> **PR #183** - [Add function HA Restarted VMs Email Notification](https://github.com/vmware-samples/vcenter-event-broker-appliance/pull/183)

I had to share my excitment on Twitter when I first contributed to the project. It's a great and unique feeling and even better when the PR got merged.

<center> {{< tweet 1216830346135842820 >}} </center>

## The HA Restart Function

I'm not going into the details on how to deploy the function, because it works basically like the [vSphere Datastore Usage Email Notification](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/datastore-usage-email) function which deployment is well explained step-by-step here: [vCenter Event Broker Appliance – Part IX – Deploying the Datastore Usage Email sample script in VMC](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/datastore-usage-email).

Exactly two files are important to deploy the function. The `vcconfig-ha-restarted-vms.json` and the `stack.yml.`
You have to fill out all the fields in the `vcconfig-ha-restarted-vms.json` file to have the required information like e.g. for the vCenter Server as well as for the SMTP server stored as a Kubernetes Secret.

```json
{
    "VC" : "vcsa.jarvis.lab",
    "VC_USERNAME" : "vebauser@jarvis.lab",
    "VC_PASSWORD" : "VMware1!",
    "SMTP_SERVER" : "smtp.gmail.com",
    "SMTP_PORT" : "587",
    "SMTP_USERNAME" : "unknownidentity@gmail.com",
    "SMTP_PASSWORD" : "securepwd",
    "EMAIL_TO": ["unknownidentity@gmail.com"],
    "EMAIL_FROM" : "Jarvis Lab"
}
```

*<center>vcconfig-ha-restarted-vms.json example</center>*

The below listed `stack.yml` file describes on which vCenter Event the function will be invoked, where the function image is stored and on which VEBA instance the function will finally run.

- `topic: com.vmware.vc.HA.ClusterFailoverActionCompletedEvent`
- `image: harbor.jarvis.lab/veba/veba-powercli-ha-restarted-vms:latest`
- `gateway: https://veba041.jarvis.lab`

```yml
version: 1.0
provider:
  name: openfaas
  gateway: https://veba041.jarvis.lab
functions:
  powercli-ha-restarted-vms:
    lang: powercli
    handler: ./handler
    image: harbor.jarvis.lab/veba/veba-powercli-ha-restarted-vms:latest
    environment:
      write_debug: true
      read_debug: true
      function_debug: false
    secrets:
      - vcconfig-ha-restarted-vms
    annotations:
      topic: com.vmware.vc.HA.ClusterFailoverActionCompletedEvent
```

*<center>stack.yml example</center>*

I have built the function from scratch and pushed it into my local container registry Harbor. If you would like to read more about *"How to push your function images to Harbor"*, take a look here: [Using Harbor with the VMware Event Broker Appliance](https://rguske.github.io/post/using-harbor-with-the-vcenter-event-broker-appliance/).

After successfully deploying the function to my VEBA the next step is to simulate a host outage to trigger the aforementioned vCenter HA event.

## ESXi Kernel Panic command

I'm using a [nested vSphere Cluster](https://www.virtuallyghetto.com/nested-virtualization) in my Homelab to simulate the outage. Connect to your nested ESXi host via `ssh` and execute the following command to perform a ESXi Kernel Panic. Please use this command with caution!

{{< admonition danger "Danger" true >}}
vsish -e set /reliability/crashMe/Panic 1
{{< /admonition >}}

And here it goes! The below screenshot from the H5 vSphere client shows the affected host (top left: nesxi67-02.jarvis.lab) as well as the occurred vCenter event which triggers our function.

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200701_023157.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200701_023157.jpg" class="center" >}}

A more detailed view can be done via a terminal. Below you can see the deployed *ha-restarted-vms* function running as a Kubernetes POD on VEBA (lower left), the ESXi Kernel Panic command (lower right) and the invocation of the *HA Restart Function* triggered by the occurred vCenter HA event (top right).

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200629_102941.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200629_102941.jpg" class="center" >}}

## You have mail!

If everything operates as expected, you should have received an email according to the following screenshot of the one I received.

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200701_022720.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200701_022720.jpg" class="center" >}}

**Thanks Bob! Keep up the great work.**

