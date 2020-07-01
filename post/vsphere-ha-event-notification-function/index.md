# VMware Event Broker Appliance - vSphere HA Event Notification Function


## CCC
When you've read through this short post and you feel encouraged to take a step beyond your comfort zone, then this post and my intention reached its goal.

### Community
I recently had the pleasure to suppport a really cool guy out of the [#VEBA Slack Community](https://vmwarecode.slack.com/archives/CQLT9B5AA), his name is Bob, with his function example contribution to the [VMware Event Broker](https://vmweventbroker.io/) project. On a remote session late at night, at least it was for me due to the different timezones CET :full_moon: and EST :sun:, Bob introduced his Powershell/ PowerCLI function to me and I was immediately thrilled about what the ƒ(x) does. It covers a use case that one of my customers brought up some time ago when I first introduced VEBA to them. Bob's engagement was really contagious and getting work done was quite fun.

### Code
Bob told me that he developed a script some time ago, which will send out a notification via Email after a vSphere HA event occurred. This Email will have the affected host mentioned as well as all affected VMs which has been restarted through vSphere HA. The problem was, that the execution of the script was a manual task every time a ESXi host outage took place...**at least until he became aware of VEBA**.

{{< admonition quote "Quote" true >}}
*"...I knew VEBA would be a prime place to attempt to do this for me."*
{{< /admonition >}}

Then he started digging into it.

There is really good material available on *"Getting started with VEBA"* like for example this [Series](http://www.patrickkremer.com/veba/) by [Patrick Kremer](https://twitter.com/KremerPatrick) as well as articles and documentation on *"Writing your own function"* like e.g. [Writing your first Serverless Function](https://medium.com/@pkblah/writing-your-first-serverless-function-23508cb4ea11) by [Partheeban Kandasamy (PK)](https://twitter.com/pkblah) and also here at the [Documentation](https://vmweventbroker.io/kb/contribute-functions) section at VEBA's homepage.

**You don't have to reinvent the wheel!** If you have great `code`, like Bob, take it, make your hands dirty and write your first function in any of your preferred programming language! It is not that complicated and I promise you the joy of success when your first ƒ(x) does exactly what you built it for.

If you don't have own code available, start looking here: [VMware {code} Sample Exchange](https://code.vmware.com/samples?categories=Sample&keywords=&tags=PowerShell%7CVMware%20PowerCLI&groups=&filters=&sort=dateDesc&page=). The community is diligent and willing to share.

### Contribution
Filing my first Pull Request on Github to an OpenSource Project was a really great and unique feeling and even better when it was merged into the project.

<center> {{< tweet 1216830346135842820 >}} </center>

And I'm sure Bob will agree to this :grin:.

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200629_040718.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200629_040718.jpg" class="center" >}}

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200630_115600.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200630_115600.jpg" class="center" >}}

Here we go and waiting for it to get merged: VMware Event Broker Appliance <i class='fab fa-github fa-fw'></i> PR #183 - [Add function HA Restarted VMs Email Notification](https://github.com/vmware-samples/vcenter-event-broker-appliance/pull/183)

## The HA Restart Function
Now let's have a quick look at the function itself.

```shell
ha-restarted-vms
├── handler
│   └── script.ps1
├── README.md
├── stack.yml
├── template
│   └── powercli
│       ├── Dockerfile
│       ├── function
│       │   └── script.ps1
│       └── template.yml
└── vcconfig-ha-restarted-vms.json
```

The function basically works like the [vSphere Datastore Usage Email Notification](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/datastore-usage-email) function which deployment is explained step-by-step here: [vCenter Event Broker Appliance – Part IX – Deploying the Datastore Usage Email sample script in VMC](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples/powercli/datastore-usage-email). 

You have to fill out all fields in the `vcconfig-ha-restarted-vms.json` to have all neccessary information for your function stored as a Kubernetes Secret.

Example:

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

The `stack.yml` describes on which vCenter Event the function will be invoked (`topic: com.vmware.vc.HA.ClusterFailoverActionCompletedEvent`), where the function image is stored (`image: harbor.jarvis.lab/veba/veba-powercli-ha-restarted-vms:latest`) and on which VEBA instance the function will finally run (`gateway: https://veba041.jarvis.lab`).

Validating the functionallity of the Pull Request is neccessary and therefore I built the function fresh from scratch (`faas-cli build -f stack.yml`) and pushed it to my local container registry Harbor (`faas-cli push -f stack.yml`). If you would like to read more about *"How to push your function images to Harbor"*, take a look here: [Using Harbor with the VMware Event Broker Appliance](https://rguske.github.io/post/using-harbor-with-the-vcenter-event-broker-appliance/).

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

After successfully deploying the function to my VEB-Appliance the next step is to simulate a host failure to trigger the aforementioned vCenter HA event.

## ESXi Kernel Panic command
I'm using a [nested vSphere Cluster](https://www.virtuallyghetto.com/nested-virtualization) in my Homelab to simulate the Host failure/ the vSphere HA event. Connect to your ESXi host via `ssh` and execute the following command to perform a ESXi Kernel Panic. Please use this command with caution!

{{< admonition danger "Danger" true >}}
vsish -e set /reliability/crashMe/Panic 1
{{< /admonition >}}

The screenshot below shows the deployed *ha-restarted-vms* function running as a Kubernetes POD on VEBA (lower left), the ESXi Kernel Panic command (lower right) and the invocation of the *HA Restart Function* triggered by the occured vCenter HA event (top right).

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200629_102941.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200629_102941.jpg" class="center" >}}

Here's the view via the H5 vSphere client:

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200701_091956.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200701_091956.jpg" class="center" >}}

## You have mail!
Let's check my :email:.

{{< image src="/img/posts/202006_harestartfunction/CapturFiles-20200629_100712.jpg" src-s="/img/posts/202006_harestartfunction/CapturFiles-20200629_100712.jpg" class="center" >}}

Nice!

**Thanks Bob!**

<center>Thanks for reading.</center>
