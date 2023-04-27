---
published: false
date: '2023-04-24 13:32 -0400'
title: ASR9k Inline MAP-T Border Relay Configuration and Troubleshooting
tags:
  - iosxr
author: Nikolai Karpyshev
excerpt: >-
  ASR9k Supports inline MAP-T Border Relay functions explained in RFC 7599. This
  tutorial covers basic configuration and advanced torubleshooting required for
  implementing this feature in CLassic (32-bit) and Evolved (64-bit) XR SW
  versions.
---
## Introduction

ASR9k supports Border Relay MAP-T function (explained in RFC 7599) without the need of a Service Module Line Card (with 4th and 5th generation of Line Cards). Please see the [Cisco documentation](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-4/cgnat/configuration/guide/b-cgnat-cg-asr9k-74x/b-cgnat-cg-asr9k-71x_chapter_0100.html#concept_7CB80766F8944515A1A2F557810AFC28) for the Details and Restrictions to configure it. 
Each MAP-T instance creates a Policy Based Routing rule which steers the traffic from ingress service-inline interface to the CGv6 application which removes the need for ISM/VSM service module. 
This Tutorial will provide step by step configuration and Trobleshooting approach to enable this feature and verify it is working correctly.




##  Scenario

In the current Example we will consider the following scenario:

![MAP-T Scenarios with ASR9k acting as Inline Border Router](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mapt_scenario.png)

Host 166.1.32.1 port 2321 in the Private IPv4 domain needs to connect to Internet IPv4 server 8.8.8.8 port 2123 through the pure IPv6 Backbone. From the connectivity perspektive IP flow 166.1.32.1 <-> 8.8.8.8 will be translated twice prior (MAPT-CE) and after (MAPT-BR) IPv6 Backbone.

We will use 3601:d01:3344::/48 subnet to translate the Internet Host address (external domain) and 2701:d01:3344::/48 for Private host translation (CPE domain). Based on the translation rules 8.8.8.8 will be transated as 

We will explore into ASR9k role as MAP-T Inline (no Service Modules) Border Router. MAP-T CE functionality is not considered in this Tutorial.

## Border Router Address Translation

In IPv4→IPv6, destination address and port are translated based on RFC 7599 and RFC 7597 and source address by RFC 6052.

In IPv6→IPv4 translation, Source address and port are translated based on RFC 7597 and RFC 7599 and destination address by RFC 6052. 

Lets examine this based on IPv4 to IPv6 translation (IPv6 to IPv4 similar in reverse direction):

1. Destination Address Translation (166.1.32.1 -> 2701:d01:3344::):

We need to define key numbers for Port-Mapping Algorythm (rfc7597). Based on our scenario config (see Configuration Section) those will be:

| Parameter         | Value     | Calculation |
|-------------------|-----------|-------------|
| k                 | 6         | 2^k = 64 (sharing ratio)        |
| m           | 3 | 2^m = 8 (contiguous-ports)|
|a|7|16-k-m|
|A|512|2^(16-a)|
|PSID|0x22|Port 2321 = 100100010001 = 100100010**001** (m=3) => 100**100010** (k=6)|
|IPv4 Suffix|00000001|32 - 24 (IPv4 prefix length) = 8 => 166.1.32.**00000001**|
|EA bit|00000001100010|IPv4 Suffix + PSID = 8 + 6 = 14 =>  00000001100010|
|Subnet|2| 64 - 48 (IPv6 prefix length) - 14 (EA bits) = 2|
|**IPv6 Address**| **2701:d01:3344:188:0:a601:2001:22**| IPv6 CPE suffix + EA BITS + Subnet  + 16 bit 0's + 32 bit ipv4 address + 16 bit PSID => 2701:d01:3344:**00000001100010** 00:0:a601:2001:**22**|


2. Source Address (8.8.8.8 -> 3601:d01:3344):
This translation is easier and defined by RFC 6052 as:

| IPv6 prefix     | v4(16) |U|(16)| suffix|
|-------------------|-----------|-------------|-------------------|-----------|

Thus
3601:d01:3344:**808:8:800::**

I'm using the traffic generator for this scenario so these are my packets:

-**IPv4 to IPv6:**

![IPv6 to IPv4](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mapt_v6.png)


-**IPv6 to IPv4:**

![IPv4 to IPv6](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mapt_v4.png)



## Configuration
In general to configure a MAP-T without service cards, one needs to perform the steps below.

SUMMARY STEPS

    configure
     service cgv6 instance-name
       service-inline interface type interface-path-id
       service-type map-t-ciscoinstance-name
         cpe-domain ipv4 prefix length value
         cpe-domain ipv6 vrf vrf-name
         cpe-domain ipv6 prefix length value
         sharing ratio <number>
         contiguous-ports <number>
         cpe-domain-name cpe-domain-name ipv4 prefix address/prefix ipv6 prefix address/prefix
         ext-domain-name ext-domain-name ipv6 prefix address/prefix ipv4-vrf vrf-name 

Within this tutorial we will focus on the following configuration and explain it in more details:

    configure
     service cgv6 CGV6-MAP-T
       service-inline interface TenGigE0/6/0/0/0
       service-type map-t-cisco MAPT-1
         cpe-domain ipv6 vrf default
         cpe-domain ipv6 prefix length 48
         cpe-domain ipv4 prefix length 24
         sharing-ratio 64
         contiguous-ports 8
         cpe-domain-name cpe1 ipv4-prefix 166.1.32.0 ipv6-prefix 2701:d01:3344::
         ext-domain-name ext1 ipv6-prefix 3601:d01:3344::/48 ipv4-vrf default


Lets verify this configuration in more details:

1.Anounce the cgv6 service and select the proper name:
**service cgv6 CGV6-MAP-T**

2.Traffic which come via the interfaces which are configured under service inline will be
subjected to MAPT rules if it matches the class map that PBR will create internally based
on CGN config. In our example 4th generation Tomahawk Line Card is used:

**service-inline interface TenGigE0/6/0/0/0**

3. Further we are entering the CPE domain to specify corresponding parameters. Please mind the domain name as it will be used in troubleshooting commands:

**service-type map-t-cisco MAPT-1**

4. Next we specify the CPE domain parameters:
- We can use either default or single non-default VRF for IPv6 traffic. After IPv4 to IPv6 translation, packet will be forwarded to that VRF:

**cpe-domain ipv6 vrf default**
   
- Next we select the the prefix length both for IPv4 and IPv6. This is needed to define if additional information required for sharing-ratio and contigious-ports which are used in port/IP translation verification (based on RFC 7599 and 7597). If IPv6 prefix length is /64 or /128 and IPv4 length is /32 then sharing-ratio and contiguous-ports will not be considered in the translation and may not to be configured.

**cpe-domain ipv6 prefix length 64**

**cpe-domain ipv4 prefix length 24**

**sharing-ratio 256**

**contiguous-ports 8**

5. Finally we configure the translation rules:
- IPv4 to IPv6 rules are defined by the cpe-domain config and after translation traffic will go out of the IPv6 VRF defined above. In particular example, traffic destined from 166.1.32.0/24 subnet will be translated to 2701:d01:3344::/48 subnet and send out VRF default (as configured in our example) based on the routing rule (see step 6 below):

**cpe-domain-name cpe1 ipv4-prefix 166.1.32.0 ipv6-prefix 2701:d01:3344::**

- IPv6 to IPv4 rules are defined based on ext-domain config. CGN will automatically derive corresponding IPv4 address from the Source and Destination addresses based on the calculation. In below example traffic towards 3601:d01:3344:5566::/48 will find the portion of IP represnting the IPv4 host and route traffic to it based on the routing rule in VRF1:

**ext-domain-name ext1 ipv6-prefix 3601:d01:3344:5566::/48 ipv4-vrf VRF1**


6. Make sure you have Routing Entry and Adjacency for the translated addresses:


In our Scenario IPv6 to IPv4 flow Destination address will translate from 


## Troubleshooting

### PBR

Verifying MAP-T we first need to make sure that corresponding PBR policies have been applied correctly. First we will check policy-map created for it:

```
		policy-map type pbr CGN_0
			handle:0x38000002
 			table description: L3 IPv4 and IPv6
 			class handle:0x78000003  sequence 1
   			  match destination-address ipv4 166.1.32.0 255.255.255.0
  			 punt service-node type cgn index 1001 app-id 0 local-id 0x1389
		   !
			class handle:0x78000004  sequence 1
			  match destination-address ipv6 3601:d01:3344::/48
 			 punt service-node type cgn index 3001 app-id 0 local-id 0x1b59
		   !
		    class handle:0xf8000002  sequence 4294967295 (class-default)
 		   !
	    end-policy-map
        `` / ``
```


- NP counters "show controller np counter <NPid> loc <LC>":
  
1. PBR intercept packets based on destination IP address, however we are also going to translate source. Thus if your source is not matching the configured entry you may see the following drops
  
  541  MDF_OPEN_NETWORK_SERVICE_PICK UNKNOWN ACTION             874220815       34715
  
One example can be in IPv6 to IPv4 translation Source address being 2701:D01:3344:4517:0:A601:2045:17 and cpe-domain rule:
cpe-domain-name cpe1 ipv4-prefix 166.1.32.0 ipv6-prefix **2701:d01:3344:4517::**
  
If we have configured IPv6 prefix length as /64 than cpe-domain address not matching the packet source: 
  2701:d01:3344:4517:: = 2701:d01:3344:4517:**0**::/64  VS 2701:D01:3344:**4517**::/64
  

- We can capture the packet on egress to verofy the translation:
  
  RP/0/RSP0/CPU0:ASR-9010-C#monitor np counter MDF_TX_WIRE.1 np0 loc 0/6/CPU0
Tue Apr 25 20:43:17.518 UTC

Usage of NP monitor is recommended for cisco internal use only.
Please use instead 'show controllers np capture' for troubleshooting packet drops in NP
and 'monitor np interface' for per (sub)interface counter monitoring

Warning: Every packet captured will be dropped! If you use the 'count'
         option to capture multiple protocol packets, this could disrupt
         protocol sessions (eg, OSPF session flap). So if capturing protocol
         packets, capture only 1 at a time.


Warning: A mandatory NP reset will be done after monitor to clean up.
         This will cause ~150ms traffic outage. Links will stay Up.
 Proceed y/n [y] > y
 Monitor MDF_TX_WIRE.1 on NP0 ... (Ctrl-C to quit)

Tue Apr 25 20:43:20 2023 -- NP0 packet

 From Fabric: 88 byte packet
0000: ac bc d9 3e 22 22 ac bc d9 3e 71 30 86 dd 60 00   ,<Y>"",<Y>q0.]`.
0010: 00 00 00 22 2c 3f 36 01 0d 01 33 44 55 66 00 08   ...",?6...3DUf..
0020: 08 08 08 00 00 00 27 01 0d 01 33 44 45 17 00 00   ......'...3DE...
0030: a6 01 20 01 00 00 11 00 00 00 00 00 00 00 08 4b   &. ............K
0040: 09 11 00 1a 57 f0 00 01 02 03 04 05 06 07 08 09   ....Wp..........
0050: 0a 0b 0c 0d 0e 0f 10 11                           ........
