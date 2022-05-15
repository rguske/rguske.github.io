# Demoing Knative Serving and Eventing using Demo-Magic-Scripts


<!--more-->

This article has been in my backlog for far too long, even though it's only a short introduction on two scripts I created to support me/you when demoing/presenting on Knative. Basically, I could stop here and just provide you the link to the repository on <i class='fab fa-github fa-fw'></i> but...it would be wrong to leave you without any context.

Also, maybe it's just right to publish the article now, as [#KubeCon](https://events.linuxfoundation.org/kubecon-cloudnativecon-europe/) 2022 Europe starts tomorrow (May, 16th) in beautiful Valencia and with it, the first [#KnativeCon](https://events.linuxfoundation.org/knativecon-europe/) 🤩

**I'm so excited about it and so happy for the project and community too.**

## What's Knative Again?

Knative is the most popular platform to run serverless workload on Kubernetes. Knative's two building blocks Serving [^1] and Eventing [^2] are providing a rich set of features in order to support you in running your serverless workload as well as to build Event-Driven Architectures (EDA). Implementation details like e.g. the Knative Pod Autoscaler [^3] (classified as reactive Pod Autoscaler) making it possible to scale your workload from n to 0(!) and vice versa. 

Also, features as Revisions [^4] and Routes [^5] making deployment models like *Blue/Green*, *Canary* and *Progressive* possible and thus providing greater flexibility in shipping your code to production. When you get to know Knative better, you'll certainly agree to Ahmet's tweet:

<center> {{< tweet user="ahmetb" id="1357124022740242433" >}} </center>

## My Knative Story

With regards to the VMware open-source project [VMware Event Broker Appliance](https://vmweventbroker.io) (short VEBA), we first started supporting Knative as an external event-processor, but fully implementing  and making it as default, was just a matter of time. In 2021, the time had come.

This was a very important step forward for the project itself, not only because it gave us many more possibilities in eventing but also in reporting all the gathered feedback internally and externally (community). [Michael Gasch](https://twitter.com/embano1), one of the maintainer of project VEBA and a great thoughtleader in this specific area of technology, wrote an awesome article on Knative which is called *[A response to "Did we market Knative wrong?"](https://www.mgasch.com/2021/06/knative-pov/)* where he's providing his insights on the projects evolvement and also on the initial adoption challenges for project VEBA itself.

Like the title of Michael's post is telling, this was his answer to the post ["Did we market Knative wrong?"](https://ahmet.im/blog/knative-positioning/) by [Ahmet Alp Balkan](https://twitter.com/ahmetb), a great member of the Knative community. I'd like to recommend you reading both articles if you got hooked on the project(s).

Enough prolouge! In order to support evangelizing both projects, Knative as well as VEBA, I needed something which supports me in talking and spreading my excitment on the two projects.

I've created two demo-scripts which will install Knative's core-components for both building blocks, the needed Channel [^6] and Broker [^7] as well the VEBA Event-Router [^8] plus an example function [^9]. Everything will be installed to an **existing** Kubernetes installation. But don't think about it as a classical installation-script (execute -> finish)! 

The idea is basically, you execute one of the scripts and it'll start typing the first command for you and stops until you execute. Therefore, you can keep focus on your presentation and continue when you are ready.

I used the handy [demo-magic.sh](https://github.com/paxtonhare/demo-magic) shell script which repository can be found on <i class='fab fa-github fa-fw'></i> as well.

## Scripts on <i class='fab fa-github fa-fw'></i>

:point_right: [Demo-Magic scripts to install and demo Knative's Serving and Eventing](https://github.com/rguske/knative-serving-eventing-demo-magic)

### Demo-Script-Recording Serving

<script id="asciicast-ovSDQy5DtjI8Z0Wg5l0VxAh9y" src="https://asciinema.org/a/ovSDQy5DtjI8Z0Wg5l0VxAh9y.js" async></script>

### Demo-Script-Recording Eventing

<script id="asciicast-PD7yafLpL7D6xVabFbnzBmSbO" src="https://asciinema.org/a/PD7yafLpL7D6xVabFbnzBmSbO.js" async></script>

## :movie_camera: VMware {code} Session CODE2762

You can also see both scripts in action in my VMworld {code} Session [#CODE2762](https://twitter.com/search?q=%23CODE2762&src=typed_query) recording.

{{< youtube ieUqfir5Oag >}}

## Resources

- [Did we market Knative wrong?](https://ahmet.im/blog/knative-positioning/) by [Ahmet Alp Balkan](https://twitter.com/ahmetb)
- [A response to "Did we market Knative wrong?"](https://www.mgasch.com/2021/06/knative-pov/) by [Michael Gasch](https://twitter.com/embano1)
- [What is Knative Serving? (A Colorful Guide)](https://tanzu.vmware.com/developer/guides/knative-serving-wi/) by [Whitney Lee](https://twitter.com/wiggitywhitney)
- [Advanced Service Concurrency and Rollouts with Tanzu](https://tanzu.vmware.com/developer/blog/advanced-service-concurrency-and-rollouts-with-tanzu/) by [Dodd Pfeffer](https://www.linkedin.com/in/doddpfeffer/)

[^1]: [Knative Serving](https://knative.dev/docs/serving/)
[^2]: [Knative Eventing](https://knative.dev/docs/eventing)
[^3]: [Knative Autosclaing](https://knative.dev/docs/serving/autoscaling/)
[^4]: [Knative Revisions](https://github.com/knative/specs/blob/main/specs/serving/knative-api-specification-1.0.md#revision)
[^5]: [Knative Routes](https://github.com/knative/specs/blob/main/specs/serving/knative-api-specification-1.0.md#route)
[^6]: [Knative Channels](https://knative.dev/docs/eventing/channels/)
[^7]: [Knative Broker](https://knative.dev/docs/eventing/broker/)
[^8]: [VEBA Event-Router](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/vmware-event-router)
[^9]: [VEBA Example Function](https://github.com/vmware-samples/vcenter-event-broker-appliance/tree/development/examples)

