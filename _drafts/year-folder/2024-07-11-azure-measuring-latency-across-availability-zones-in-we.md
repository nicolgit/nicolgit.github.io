---
title: Measuring latency between Azure Availabity Zones and the impact of an NVA in between V2
date: 2024-07-11 10:00
tags: [Azure, networking, hub-and-spoke, latency, availability-zone, azure firewall, peering, Proximity Placement Group, Virtual Network Gateway]
excerpt: "In this post I measure the latency between virtual machines in various network configurations on Azure"

header:
  overlay_image: https://live.staticflickr.com/65535/52075200779_53b4400eae_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/52075200779)"
---

[Latency](https://www.techtarget.com/whatis/definition/latency) is an expression of how much time it takes for a data packet to travel from one designated point to another. Ideally, latency will be as close to zero as possible. High network latency can **dramatically increase webpage load times**, interrupt video and audio streams, and render an application unusable. Depending on the application, even a relatively small increase in latency can ruin UX.

[Azure availability zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview) are physically separate locations within each Azure region that are tolerant to local failures. **Azure availability zones are connected by a high-performance network with a round-trip latency of less than 2ms**. 

in Azure, each data center is assigned to a physical (availability) zone. Physical zones are mapped to logical (availability) zones in your Azure subscription. You can design resilient solutions by using Azure services that use availability zones. Co-locate your compute, storage, networking, and data resources across an availability zone, and replicate this arrangement in other availability zones.

A [proximity placement group](https://azure.microsoft.com/en-us/blog/announcing-the-general-availability-of-proximity-placement-groups/) is an Azure Virtual Machine logical grouping capability that you can use to decrease the inter-VM network latency associated with your applications. When the VMs are deployed within the same proximity placement group, they are physically located as close as possible to each other.

Understanding the latency implications of different network configurations is essential when designing a high-availability and resilient architecture. It is crucial to consider the impact of latency on application performance and user experience. By measuring latency between virtual machines in various network setups, you can assess the effectiveness of different configurations. This knowledge can help you make informed decisions when building your architecture.

In this blog post I measure the latency between virtual machines deployed in Azure's **West Europe region** in the following configurations:

* same v-net, same availability zone, same proximty placement group
* same v-net, across availability zones
* multiple v-nets (in peering), same availability zone
* multiple v-nets (in peering), across availability zones
* multiple v-nets connected in a Hub & Spoke topology and Routing via [Azure Virtual Network Gateway](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways)
* multiple v-nets connected in a Hub & Spoke topology and Routing via [Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview)

Rather than the absolute values, my focus is on assessing the impact in terms of latency of various network configurations.

To measure latency, I used [Latte](https://github.com/microsoft/latte), a Windows network latency benchmark tool available on GitHub. You can find more information on how to measure network latency between Azure VMs using Latte (on Windows) and SOCKPERF (on Linux) at [this link](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-test-latency?tabs=windows ).

> The [**Azure hub and spoke playground**](https://github.com/nicolgit/hub-and-spoke-playground) is a GitHub repository where you can find a reference network architecture that I use as a common base to implement configurations and test networking and connectivity scenarios. I have also used it here as a starting point to build the lab used in this post.

# Scenario 1 - one virtual network

In this scenario I have a calling machine in one availability zone and 3 additional machines each in a different availability zone. In availability zone 1 I have also placed both machines in the same proximity placement group to have the best possible latency, as shown below.

![same network](../../assets/post/2022/latency-scenario-1.png)

Here the measures from `spoke01-az-01` (Availability Zone 1).

| commamd | Availability Zone |  Latency (usec) |  
|---------|-------------------|-----------------|
`latte -c -a 10.13.1.6:80 -i 60000` | 1  | **65.15**    |
`latte -c -a 10.13.1.7:80 -i 60000` | 2  | **119.87**   uhm|
`latte -c -a 10.13.1.8:80 -i 60000` | 3  | **1199.54**  |

Takeaways:

* The Proximity Placement Group performs exceptionally well, with latency well below 2ms (only 65 microseconds!)
* When communicating between availability zones, I consistently measured an average latency below 2ms.
* The significant difference in latency between communication from AZ1 to AZ2 and AZ1 to AZ3 is likely due to the non-uniform distance between the three Availability Zones in the West Europe region. These three zones are positioned in a highly pronounced isosceles triangle around Amsterdam.

# Scenario 2 - two virtual networks in peering

In this scenario I have measured the impact of a network peering. I have created 3 more machines, in 3 availability zones, on another network, in peering.

![peering](../../assets/post/2022/latency-scenario-2.png)

Here the measures from `spoke01-az-01` (availability zone 1) to machines in another virtual network in peering.

| commamd | Availability Zone |  Latency (usec) |  
|---------|-------------------|-----------------|
`latte -c -a 10.13.2.5:80 -i 60000` | 1  | **57.92**   |
`latte -c -a 10.13.2.6:80 -i 60000` | 2  | **112.22**  |
`latte -c -a 10.13.2.7:80 -i 60000` | 3  | **1062.29** |

Takeaways:
* Network peering does not add any measurable overhead.
* The latency between two VMs in AZ1, even if they are not in the same proximity group, is almost equal to or even better than the latency between two VMs in the same proximity group. It is possible that `spoke-02-az-01` has been provisioned very close to `spoke-01-az-01`, although it is not guaranteed.

# Scenario 3 - two virtual networks in H&S configuration with a Virtual Network Gateway in between

In this scenario I moved to a more classic configuration: I eliminated peering and routed traffic through a central hub and an Azure Virtual Network Gateway.

![hub-and-spoke](../../assets/post/2022/latency-scenario-3-4.png)

Here the measures from `spoke01-az-01` (availability zone 1) to machines in another virtual network via an Azure Virtual Network Gateway in the Hub Network.

| Command | Av Zone |  Snt |  Last |  Avg | Best | Wrst | StDev
|---|--------|------|-------|------|------|------|------|
| `mtr 10.13.2.5` | 1  |  47 |   4.2 |  **3.4** |  1.6 | 24.4 |  4.2 |
| `mtr 10.13.2.6` | 2  |  35 |   2.9 |  **4.3** |  2.8 | 12.3 |  2.6 |
| `mtr 10.13.2.7` | 3  |  36 |   3.4 |  **4.5** |  2.8 | 15.7 |  3.1 |

Takeaways

* Latency increased up to **3.4/4.5ms**, because each packet have to cross 2 peerings and a virtual appliace (Azure Virtual Network Gateway)
* **Staying in the same availability zone also have a positive impact on latency**: an average of +1ms in cross availability zone topology is not huge in se, but is a relative increase quite high (+50%)

# Scenario 4 - two virtual networks in H&S configuration with an Azure Firewall in between

In this last scenario I implemented the [reference architecture described in the cloud adoption framework](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke), that is a hub and spoke, with an Azure Firewall to control all the traffic. 

![hub and spoke](../../assets/post/2022/latency-scenario-3-4.png)

Here the measures from `spoke01-az-01` (availability zone 1) to machines in another virtual network and different availability zones, via Azure Firewall in the Hub Network.

| Command | Av Zone |  Snt |  Last |  Avg | Best | Wrst | StDev |
|---|--------|------|-------|------|------|------|------|
| `mtr 10.13.2.5` | 1  |  58   | 2.4  | **2.3**  | 1.8  | 5.7  | 0.6 |
| `mtr 10.13.2.6` | 2  |  34   | 3.4  | **3.1**  | 2.6  | 4.3  | 0.3 |
| `mtr 10.13.2.7` | 3  |  30   | 3.7  | **3.4**  | 2.9  | 3.9  | 0.3 |

Takeaways

* Latency is far better than an Azure Virtual Network Gateway with Azure Firewall in between
* **When source and destination are in the same availability zone it has almost the same as for a simple peering**
* When source and destination are **in different availabity zones, latency grows up to 3.1/3.4ms**

# Final thoughts

* Where possible use **always Proximity Placement Groups** to have sub 2ms latency
* Traffic between zones is confirmed to be **on the order of 2ms**
* Peerings and hub-and-spoke **have no real impac**t if a Firewall is used in the middle
* I would avoid to use a VPN Gateway for **cross spokes communications**

More information
* Test Network Latency on Azure VM: https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-test-latency?tabs=windows 
* Latte repo: https://github.com/microsoft/latte
* Azure Availability Zone explained: https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview?tabs=azure-cli 