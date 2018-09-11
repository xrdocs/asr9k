---
author: Cisco Web Team
published: true
date: '2018-09-06 13:47 -0700'
title: Dynamic Packet Buffering
position: top
excerpt: >-
  The ASR9000 4th Generation Ethernet linecards use an intelligent packet
  buffering technique to provide high performance deep buffering.
---


{% include toc %}


![AU36244.jpg]({{site.baseurl}}/images/AU36244.jpg)


# Intelligent Packet Buffering on ASR9000 4th Generation Ethernet Line cards



## Document Purpose


The ASR9000 4th Generation Ethernet linecards use an intelligent packet buffering technique to provide high performance deep buffering. Such technique takes advantage of on-demand buffer allocation for efficient handling of instantaneous traffic bursts and it is optimized for high bandwidth interfaces on dense linecards. This paper provides an introduction to this new dynamic buffer management system and explains its strengths over the static buffer allocation models.
 
Packet buffers are designed to absorb high-speed incoming temporary data bursts, to avoid packet losses and manage latency. For such scenarios, dynamic buffer management techniques are more efficient than static allocations, as they allow assignment of available buffers to physical interfaces on an interim basis only. They become even more relevant in the case of high speed interfaces due to the amount of memory required to buffer traffic even for very short periods of time. Static allocations, on the contrary, are useful as a fallback option for very specific use cases where per-priority-buffer limits and per-port bandwidth guarantee are more important.

The following sections will describe both mechanisms as they are supported on the 4th Generation linecards for the ASR9000 product family.


### High Bandwidth buffering and Model Shifting


Service provider edge (PE) router plays a crucial role in the network, aggregating large quantity of traffic requiring network service handling (e.g. business L2 or L3VPN) from tens to hundreds of access devices. Therefore, the aggregation capacity of a PE router, in terms of number of access devices per port, is high. Simultaneously, each of those devices, expects to receive large volumes of downstream data from content delivery and service providers over the core network. That can often result in congestion at the PE due to bulk of traffic waiting to be delivered into the access network. There can be instantaneous traffic burst or even periods of sustained network congestions on the edge routers. As a result, the packet-buffering model must find efficient ways to carve memory during congestion in any given subset of ports, while ensuring fairness across all ports and honoring the different traffic priorities.

The dramatic increase in forwarding throughput and port densities in the recent years, coupled with the growing aggregation scale on converged network elements, has pushed the boundaries on achievable packet processing speeds and per-packet memory access performances. Integrating very large packet buffer memories with the super-high throughput network processor (NP) is prohibitive from a cost and power perspective. That is due to asymmetric technology improvements in NP throughput growth vs. commodity memory performance. As a rule of thumb, edge routers require 50ms of line-rate output queue buffers. This translates to 625 MB of high-speed packet buffering for a single 100G interface, compared to the mere 6.25 MB for 1G ports. 

![Picture2.png]({{site.baseurl}}/images/Picture2.png)

Figure 1.1 Interface Bandwidth and Buffer Space Demanding
{: .text-center}

Figure 1.1 shows how the increase in port bandwidth has created an exponential demand for buffer space. These changes are driving more efficient and flexible packet buffer allocation models on ASR9000 4th Generation linecards. 

Moreover, ASR9000 4th Generation Ethernet linecards employ a new memory technology known as High Bandwidth Memory (HBM), which provides high performance memory bandwidth at decreased power consumption. The HBM memory has multiple access channels that can be used simultaneously by different NP components, resulting in a very high memory access throughput.
![image-center]({{site.baseurl}}/images/rsp5s.png)

### Intelligent dynamic packet buffer management


With Ethernet interface speeds transitioning from 10Gbps to 100Gbps or even 400Gbps, the demand for packet buffering has exploded to hundreds of GBs of high bandwidth memory. This is several orders of magnitude higher than what the current memory technologies can provide at reasonable price and power consumption. The ASR9000 4th Generation Ethernet linecards are at the forefront of this transition, providing a dynamic, shared packet buffer design that replaces the less efficient and rigid static allocation.

The new solution enables the HBM to maximize burst absorption through on-demand buffer allocation, while sharing resources among all network ports that are part of the same Network Processor (NP). As an example, this enables an HBM memory that could only provide 50 ms of per-port buffering in a static buffer management model to buffer up to 100ms, when not all ports on the NP are congested. Moreover, the 4th Generation NP offers flexible programming capabilities that ensure:

- priority protection 

- protection of critical control traffic, 

- guaranteed 10 ms of port buffering on all ports 

- dynamic buffer management for the remaining transit traffic.


Due to the better burst absorption capabilities and efficient sharing of high performance memory resources across NP ports, the ASR9000 4th Generation Ethernet linecards implement dynamic buffer management by default. 


This dynamic buffer allocation model manages the 3GB of HBM memory available for packet buffering by defining 3 logical regions:

- **Shared region** – 2.5 GB of HBM is available for shared packet buffering amongst all ports of the NP. This region is consumed first on a first-come and first-serve basis by those ports.

- **Protection region** - 300 MB of HBM is available for port guarantees and for priority traffic protection. The only ports that have not exhausted their guaranteed 10ms of buffering are allowed to use this region. This region is also used for ports that have not exhausted their 100ms limit to buffer transit high priority packets. Transit priority packets are scheduled first out of any port; so, priority buffering is rare and happens only for transient bursts on a well-designed network.

- **Reserved region** - The remaining 200 MB is reserved for internal ports and critical control protocol protection. This region is not available for transit packet buffering.

Figure 2.1 illustrates few use cases of how the dynamic buffer management at the NP allocates buffers for less, average and high congestion scenarios. Congestion levels are defined by the number of ports at the NP experiencing congestion. As you can see, the three regions together provide a flexible mechanism to share buffers while still providing some degree of protection to priority and per-port buffering. Diagnostics and critical protocol traffic have their own reserved region for hitless prioritization even during maximum NP congestion.

![Picture3.png]({{site.baseurl}}/images/Picture3.png)

Figure 2.1 Dynamic buffer management scenarios
{: .text-center}


### Static buffer management innovations


Conventional static buffer management techniques allocate buffers on a per-port or per-port-group basis, thus exclusively guaranteeing a predefined and fixed amount of buffer memory for each port under traffic congestion. Once this buffer carving is done during NP initialization, regions allocated to different ports cannot be leveraged to relieve congestion on an affected interface. The static reservation, therefore, trades off deterministic, guaranteed behaviors with inefficient sharing.

Static buffer allocation ensures the amount of queuing capacity determines the amount of instantaneous oversubscription a router interface is able to tolerate without dropping packets and causing performance degradation. The absolute port buffer can build the guaranteed per-priority-buffer limits up to 8 priorities queuing and ensure network oversubscription budgeting. The latter allows edge routers to be configured to absorb bursts in lieu of downstream access devices that have very low buffers.

While dynamic buffer management is efficient in addressing the buffer memory considerations on high speed interfaces described earlier, there could still be situations where a guaranteed model is preferred. The fallback option provides parity behavior with previous generation NPs, the 4th generation ASR9000 Ethernet line cards provide a configuration knob to set buffer management to static mode where required. While in static mode, one can also configure the per chunk buffer limits (show in Figure3.1) on per NP basis to customize size of packet buffer associated to each of the 100GE interfaces. This allows flexible uneven packet buffer allocation per interface depending on different traffic pattern on each interface. This allows extended flexibility to map additional buffer resources to port chunks that are expected to experience frequent congestion based on network design. The line cards can function with mix mode NPs, as different port groups on a line card may have different QoS service level agreement (SLA) requirements.

![Picture4.png]({{site.baseurl}}/images/Picture4.png)
Figure 3.1 Static or Sliced Buffering Per NP
{: .text-center}

Figure 3.1 illustrates the two options available when a user explicitly configures the NP to operate in a static buffer management mode. This mode still reserves 200 MB of HBM memory for internal ports and critical protocol protection. Therefore, only the “shared” and “protection” regions can be carved for static buffer management.

When the user configures “hw-module loc \<location\> qos port-buffer-static-limit np \<np\>”, the remaining 2.8 GB of HBM are equally carved between the four 100G NP chunks (either 100G ports or breakouts). However, the user can additionally include a chunk keyword and specify the amount of buffering to be allocated to a particular chunk. The amount of buffering that can be assigned to a given chunk ranges from 10 ms to 120 ms. The remaining 2.8 GB, minus the carved-out space for that one chunk, is equally distributed among the remaining chunks. 

As an example, without any specific per-chunk carving, each chunk gets 700 MB of memory, which translates to 50ms of buffering. However, if the one chunk is configured for 120 ms buffering, the remaining three chunks will only get 35 ms each.

This enhancement to conventional static buffering allows for a very granular control of memory allocation. It allows absolute port buffer which can ensure QoS per-priority-buffer limits control, so that the top priority queue gets to buffer up to the max port buffer limit, while the remaining priority queues appropriately spaced out from the second up to eight. Furthermore, since the configure knob on a per-chunk basis, it gives the flexibility to easily compatible to different port speeds and future card variants.

### Summary

This paper highlighted the need for change in the interface buffer allocation mechanisms and demonstrated how the 4th generation NP on the Cisco ASR9000 product family provides an innovative, dynamic, adaptive and on-demand architecture for packet buffering. In addition, the fallback option of static buffer management, the support for mixed mode NPs on the same line card and the enhancements of chunk-based user defined buffer carving should allow backward compatibility where needed.
