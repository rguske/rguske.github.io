# Quickly Recover a Broken Avi Controller

## Introduction





{{< image src="/img/posts//" caption="Figure I: " src-s="/img/posts//" >}}


{{< admonition info "" true >}}

{{< /admonition >}}



| | | |
|:---: |:---: |:---: | :---: |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |


{{< mermaid >}}
graph LR;
    subgraph HAProxy Appliance
    A[ Management ]
    B[ Workload ]
    C[ Frontend / VIPs ]
    end
    subgraph Traffic from
    G( Client / Service )
    end
    subgraph Load Balancer VIPs
    E[ Supervisor Cluster VMs ]
    F[ Guest Cluster VMs ]
    end
    A & B & C --- E
    B & C --- F
    G -.- C -.-> F & E
{{< /mermaid >}}
