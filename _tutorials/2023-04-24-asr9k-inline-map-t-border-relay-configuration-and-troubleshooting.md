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

BLA-BLA-BLA



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
         sharing rationumber
         contiguous-portsnumber
         cpe-domain-name cpe-domain-name ipv4 prefix address/prefix ipv6 prefix address/prefix
         ext-domain-name ext-domain-name ipv6 prefix address/prefix ipv4-vrf vrf-name 

Within this tutorial we will focus on the following configuration and explain it in more details:

    configure
     service cgv6 CGV6-MAP-T
       service-inline interface TenGigE0/6/0/0/0
       service-type map-t-cisco MAPT-1
         cpe-domain ipv6 vrf default
         cpe-domain ipv6 prefix length 64
         cpe-domain ipv4 prefix length 24
         sharing-ratio 256
         contiguous-ports 8
         cpe-domain-name cpe1 ipv4-prefix 166.1.32.0 ipv6-prefix 2701:d01:3344::
         ext-domain-name ext1 ipv6-prefix 3601:d01:3344:5566::/64 ipv4-vrf VRF1


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
- IPv4 to IPv6 rules are defined by the cpe-domain config and after translation traffic will go out of the IPv6 VRF defined above. In particular example, traffic destined from 166.1.32.0/24 subnet will be translated to 2701:d01:3344::/64 subnet and send out VRF default (as configured in our example) based on the routing rule (see step 6 below):

**cpe-domain-name cpe1 ipv4-prefix 166.1.32.0 ipv6-prefix 2701:d01:3344::**

- IPv6 to IPv4 rules are defined based on ext-domain config. CGN will automatically derive corresponding IPv4 address from the Source and Destination addresses based on the calculation. In below example traffic towards 3601:d01:3344:5566::/64 will find the portion of IP represnting the IPv4 host and route traffic to it based on the routing rule in VRF1:

**ext-domain-name ext1 ipv6-prefix 3601:d01:3344:5566::/64 ipv4-vrf VRF1**


6. Make sure you have Routing Entry and Adjacency for the translated addresses:


BLA-BLA-BLA











