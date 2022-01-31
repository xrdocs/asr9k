---
published: true
date: '2022-01-21 10:45 +0100'
title: ' Troubleshoot Slow BGP Convergence Due to Suboptimal Route Policies on IOS-XR'
position: hidden
tags:
  - iosxr
  - BGP
  - Troubleshooting
author: Vladimir Deviatkin
excerpt: >-
  This document describes how to diagnose slow BGP convergence issue on Cisco
  IOS® XR routers which happens because of non optimal route policies.
---
{% include toc icon="table" title="Troubleshoot Slow BGP Convergence Due to Suboptimal Route Policies on IOS-XR" %}

# Introduction

This document describes how to diagnose slow BGP convergence issue on Cisco IOS® XR routers which happens because of non optimal route policies.

# Background

BGP Convergence time consists of a number of factors. One of them is the time to process ingress or egress BGP updates by configured route policies. There are multiple ways to write a route policy to do a specific task. An optimal way helps to improve BGP convergence, minimize potential traffic drops, and avoid temporary routing loops. Cisco IOS® XR includes a profiling tool which measures time spent by specific route policy in order to estimate its processing time.

# Problem

Slow BGP Convergence is sometimes the result of route policies written in a non optimal way.

# Solution

There is an policy profiling tool for route policies which can be used without impact on performance in order to measure the time spent in each statement of a route policy at a specific attach point. You can check the run time of the route policy at this specific attach point. By default, the profiling is enabled only for aggregate route policy stats.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
router bgp 65000
neighbor 10.0.54.6
remote-as 65000
update-source Loopback0
address-family ipv4 unicast
route-policy INGRESS-ROUTE-POLICY in
!
neighbor 10.0.54.11
remote-as 65001
ebgp-multihop 255
update-source Loopback0
address-family ipv4 unicast
route-policy EGRESS-ROUTE-POLICY out
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-in-dflt default-IPv4-Uni-10.0.54.6 policy profile
Policy profiling data
Policy : INGRESS-ROUTE-POLICY
Pass : 1440233
Drop : 0
# of executions : 1440233
<mark>Total execution time : 57095msec</mark>

RP/0/RSP1/CPU0:XR1#show bgp ipv4 unicast neighbors 10.0.54.11 | i Update group
Update group: 0.3 Filter-group: 0.5 No Refresh request being processed
RP/0/RSP1/CPU0:XR1#

RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-out-dflt default-IPv4-Uni-UpdGrp-0.3-Out policy profile
Policy profiling data
Policy : EGRESS-ROUTE-POLICY
Pass : 726751
Drop : 0
# of executions : 726751
<mark>Total execution time : 108099msec</mark>
</code>
</pre>
</div>

You see the cumulative amount of time spent in order to process INGRESS-ROUTE-POLICY and EGRESS-ROUTE-POLICY.  
The profiling can be applied for ingress or egress route policy for any attach point.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 ?
  debug-policy                 Attachpoint name
  permnet                      Attachpoint name
  import                       Attachpoint name
  export                       Attachpoint name
  interafi-import              Attachpoint name
  source-rt                    Attachpoint name
  interafi-export              Attachpoint name
  retain-rt                    Attachpoint name
  addpath                      Attachpoint name
  neighbor-in-dflt             Attachpoint name
  neighbor-in-vrf              Attachpoint name
  neighbor-out-dflt            Attachpoint name
  neighbor-out-vrf             Attachpoint name
  orf-dflt                     Attachpoint name
  orf-vrf                      Attachpoint name
  dampening-dflt               Attachpoint name
  dampening-vrf                Attachpoint name
  default-originate-dflt       Attachpoint name
  default-originate-vrf        Attachpoint name
  clear-policy                 Attachpoint name
  show-policy-node0_RSP1_CPU0  Attachpoint name
  aggregation-dflt             Attachpoint name
  aggregation-vrf              Attachpoint name
  nexthop                      Attachpoint name
  allocate-label               Attachpoint name
  label-mode                   Attachpoint name
  l2vpn-import                 Attachpoint name
  l2vpn-export                 Attachpoint name
  redistribution-dflt          Attachpoint name
  redistribution-vrf           Attachpoint name
  rib-install-dflt             Attachpoint name
  rib-install-vrf              Attachpoint name
  network-dflt                 Attachpoint name
  network-vrf                  Attachpoint name
  redistribution-dflt          Attachpoint name
  redistribution-vrf           Attachpoint name
  rib-install-dflt             Attachpoint name
  rib-install-vrf              Attachpoint name
  network-dflt                 Attachpoint name
  network-vrf                  Attachpoint name
  l2vpn-export-mp2mp           Attachpoint name
  l2vpn-export-vfi             Attachpoint name
  l2vpn-export-evi             Attachpoint name
  l2vpn-export-mspw            Attachpoint name
  l2vpn-export-instance        Attachpoint name
  WORD                         Attachpoint name
RP/0/RSP1/CPU0:XR1#
</code>
</pre>
</div>

You can clear stats as needed:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#clear pcl protocol bgp speaker-0 neighbor-in-dflt default-IPv4-Uni-10.0.54.6 policy profile
RP/0/RSP1/CPU0:XR1#clear pcl protocol bgp speaker-0 neighbor-out-dflt default-IPv4-Uni-UpdGrp-0.3-Out policy profile
</code>
</pre>
</div>

If you enable debug pcl profile detail, then you get detailed stats per route policy entry.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#debug pcl profile detail
</code>
</pre>
</div>

These outputs were collected after the full Internet BGP table scale was received and propagated further.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-in-dflt default-IPv4-Uni-10.0.54.6 policy profile
Policy profiling data
Policy : INGRESS-ROUTE-POLICY
Pass : 720100
Drop : 0
# of executions : 720100
<mark>Total execution time : 222788msec !!!! about 3.7 minutes to process ingress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720100     <mark>221796msec</mark>  if as-path aspath-match ... then
                                       &lt;truePath&gt;
PXL_0_3          3525          3msec    set local-preference 150
                 3525          0msec    &lt;end-policy/&gt;
                                       &lt;/truePath&gt;
                                       &lt;falsePath&gt;
PXL_0_2        716575        225msec    set local-preference 50
               716575         82msec    &lt;end-policy/&gt;
                                       &lt;/falsePath&gt;


RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-out-dflt default-IPv4-Uni-UpdGrp-0.3-Out policy profile
Policy profiling data
Policy : EGRESS-ROUTE-POLICY
Pass : 720105
Drop : 0
# of executions : 720105
<mark>Total execution time : 221975msec  !!!! about 3.7 minutes to process egress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720105       3005msec  if as-path aspath-match ... then
                                       &lt;truePath&gt;
PXL_0_5             0          0msec    set med 70
                    0          0msec    &lt;end-policy/&gt;
                                       &lt;/truePath&gt;
                                       &lt;falsePath&gt;
PXL_0_2        720105     <mark>218008msec</mark>    if as-path aspath-match ... then
                                         &lt;truePath&gt;
PXL_0_3            25          0msec      set med 80
                   25          0msec      &lt;end-policy/&gt;
                                         &lt;/truePath&gt;
                                         &lt;falsePath&gt;
PXL_0_4        720080        145msec      set med 90
               720080         76msec      &lt;end-policy/&gt;
                                         &lt;/falsePath&gt;
                                       &lt;/falsePath&gt;

RP/0/RSP1/CPU0:XR1# 
</code>
</pre>
</div>

As you can see, line PXL_0_1 for INGRESS-ROUTE-POLICY and PXL_0_2 for EGRESS-ROUTE-POLICY are especially time consuming and slow the convergence down.  
If you correlate them with the route policies, then you can see that AS-PATH-SET-11 in INGRESS-ROUTE-POLICY and AS-PATH-SET-22 in EGRESS-ROUTE-POLICY cause the problem. Each AS path set is made of 100 regular expression lines which is a pretty inefficient way to write a policy since it does not leverage any possible optimization or the power of regex.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
route-policy INGRESS-ROUTE-POLICY
  if as-path in AS-PATH-SET-11 then
    set local-preference 150
  else
    set local-preference 50
  endif
end-policy


route-policy EGRESS-ROUTE-POLICY
  if as-path in AS-PATH-SET-21 then
    set med 70
  elseif as-path in AS-PATH-SET-22 then
    set med 80
  else
    set med 90
  endif
end-policy


as-path-set AS-PATH-SET-11
  ios-regex '^65101 65201_',
  ios-regex '^65102 65202_',
  ios-regex '^65103 65203_',
  ios-regex '^65104 65204_',
  ios-regex '^65105 65205_',
 --- removed 90 similar lines ---
  ios-regex '^65195 65295_',
  ios-regex '^65196 65296_',
  ios-regex '^65197 65297_',
  ios-regex '^65198 65298_',
  ios-regex '^65199 65299_'
end-set

as-path-set AS-PATH-SET-21
  ios-regex '^$'
end-set

as-path-set AS-PATH-SET-22
  ios-regex '^65169(_65169)*$',
  ios-regex '^65392(_65392)*$',
  ios-regex '^65133(_65133)*$',
  ios-regex '^65231(_65231)*$',
  ios-regex '^65161(_65161)*$',
 --- removed 90 similar lines ---
  ios-regex '^65281(_65281)*$',
  ios-regex '^65336(_65336)*$',
  ios-regex '^65238(_65238)*$',
  ios-regex '^65381(_65381)*$',
  ios-regex '^65103(_65103)*$'
end-set
</code>
</pre>
</div>

In order to improve policy performance, you can evaluate configuration of AS Path sets with native as-path match operation instead of regular expression. Alternatively, you can use regular expressions inside route policies in a collapsed manner hence reduce the number of regular expression lines used.  
This table lists AS path match criteria offered by Route Policy Language (RPL). The native matching functions use a binary matching algorithm which offers better performance compared to the regular expression match engine. Most of the common ios-regex match scenarios could be written with them (or their combinations).

| Command Syntax | Description |
|-|-|
| **is-local** | Determines if the router (or another router within this autonomous system or confederation) originated the route |
| **length** | Performs a conditional check based on the length of the AS path |
| **neighbor-is** | Tests the autonomous system number or numbers at the head of the AS path against a sequence of one or more integral values or parameters. |
| **originates-from** | Tests an AS path against the AS sequence from the start with the AS number that originated a route. |
| **passes-through** | Tests to learn if the specified integer or parameter appears anywhere in the AS path or if the sequence of integers and parameters appears. |
| **unique-length** | Performs specific checks based on the length of the AS path ignoring duplicates |



# Verification

These are rearranged route policies written with the help of native match criteria. This leads to substantially reduced processing time.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
route-policy INGRESS-ROUTE-POLICY
  if as-path in AS-PATH-SET-11 then
    set local-preference 150
  else
    set local-preference 50
  endif
end-policy

route-policy EGRESS-ROUTE-POLICY
  if as-path is-local then
    set med 70
  elseif as-path in AS-PATH-SET-22 and as-path unique-length is 1 then
    set med 80
  else
    set med 90
  endif
end-policy


as-path-set AS-PATH-SET-11
  neighbor-is '65101 65201',
  neighbor-is '65102 65202',
  neighbor-is '65103 65203',
  neighbor-is '65104 65204',
  neighbor-is '65105 65205',
--- removed 90 similar lines ---
  neighbor-is '65195 65295',
  neighbor-is '65196 65296',
  neighbor-is '65197 65297',
  neighbor-is '65198 65298',
  neighbor-is '65199 65299'
end-set

as-path-set AS-PATH-SET-22
  originates-from '65169',
  originates-from '65392',
  originates-from '65133',
  originates-from '65231',
  originates-from '65161',
--- removed 90 similar lines ---
  originates-from '65281',
  originates-from '65336',
  originates-from '65238',
  originates-from '65381',
  originates-from '65103'
end-set
</code>
</pre>
</div>

These outputs are collected after the full Internet BGP table scale is received and propagated further.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-in-dflt default-IPv4-Uni-10.0.54.6 policy profile
Policy profiling data
Policy : INGRESS-ROUTE-POLICY
Pass : 720100
Drop : 0
# of executions : 720100
<mark>Total execution time : 9612msec !!!! about 10 seconds to process ingress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720100       8540msec  if as-path aspath-match ... then
                                       &lt;truePath&gt;
PXL_0_3          7128          2msec    set local-preference 150
                 7128          1msec    &lt;end-policy/&gt;
                                       &lt;/truePath&gt;
                                       &lt;falsePath&gt;
PXL_0_2        712972        276msec    set local-preference 50
               712972         80msec    &lt;end-policy/&gt;
                                       &lt;/falsePath&gt;


RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-out-dflt default-IPv4-Uni-UpdGrp-0.3-Out policy profile
Policy profiling data
Policy : EGRESS-ROUTE-POLICY
Pass : 720126
Drop : 0
# of executions : 720126
<mark>Total execution time : 12399msec !!!! about 12 seconds to process egress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720126        190msec  if as-path is-local then
                                       &lt;truePath&gt;
PXL_0_7             0          0msec    set med 70
                    0          0msec    &lt;end-policy/&gt;
                                       &lt;/truePath&gt;
                                       &lt;falsePath&gt;
PXL_0_2        720126      11190msec    if as-path aspath-match ... then
                                         &lt;truePath&gt;
PXL_0_4        262734         65msec      if as-path unique-length is 1 then
                                           &lt;truePath&gt;
PXL_0_5            25          0msec        set med 80
                   25          0msec        &lt;end-policy/&gt;
                                           &lt;/truePath&gt;
                                           &lt;falsePath&gt;
PXL_0_6        720101        164msec        set med 90
               720101         57msec        &lt;end-policy/&gt;
                                           &lt;/falsePath&gt;
                                         &lt;/truePath&gt;
                                         &lt;falsePath&gt;
                                          &lt;reference&gt;
GOTO :                                     PXL_0_6
                                          &lt;/reference&gt;
                                         &lt;/falsePath&gt;
                                       &lt;/falsePath&gt;


RP/0/RSP1/CPU0:XR1# 
</code>
</pre>
</div>

Alternatively, ios-regex lines could be collapsed. This also helps to improve the performance.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
route-policy INGRESS-ROUTE-POLICY
  if as-path in (ios-regex '^(65101_65201|65102_65202|65103_65203|65104_65204|65105_65205|65106_65206|65107_65207|65108_65208|65109_65209|65110_65210)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65111_65211|65112_65212|65113_65213|65114_65214|65115_65215|65116_65216|65117_65217|65118_65218|65119_65219|65120_65220)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65121_65221|65122_65222|65123_65223|65124_65224|65125_65225|65126_65226|65127_65227|65128_65228|65129_65229|65130_65230)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65131_65231|65132_65232|65133_65233|65134_65234|65135_65235|65136_65236|65137_65237|65138_65238|65139_65239|65140_65240)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65141_65241|65142_65242|65143_65243|65144_65244|65145_65245|65146_65246|65147_65247|65148_65248|65149_65249|65150_65250)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65151_65251|65152_65252|65153_65253|65154_65254|65155_65255|65156_65256|65157_65257|65158_65258|65159_65259|65160_65260)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65161_65261|65162_65262|65163_65263|65164_65264|65165_65265|65166_65266|65167_65267|65168_65268|65169_65269|65170_65270)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65171_65271|65172_65272|65173_65273|65174_65274|65175_65275|65176_65276|65177_65277|65178_65278|65179_65279|65180_65280)') then
    set local-preference 150
  endif
  if as-path in (ios-regex '^(65181_65281|65182_65282|65183_65283|65184_65284|65185_65285|65186_65286|65187_65287|65188_65288|65189_65289|65190_65290)') then
    set local-preference 150
  else
    set local-preference 50
  endif
end-policy

route-policy EGRESS-ROUTE-POLICY
  if as-path in (ios-regex '^$') then
    set med 70
  endif
  if as-path in (ios-regex '^65169(_65169)*$|^65392(_65392)*$|^65133(_65133)*$|^65231(_65231)*$|^65161(_65161)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65354(_65354)*$|^65331(_65331)*$|^65342(_65342)*$|^65295(_65295)*$|^65208(_65208)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65149(_65149)*$|^65350(_65350)*$|^65115(_65115)*$|^65300(_65300)*$|^65322(_65322)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65102(_65102)*$|^65329(_65329)*$|^65237(_65237)*$|^65218(_65218)*$|^65153(_65153)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65263(_65263)*$|^65116(_65116)*$|^65112(_65112)*$|^65114(_65114)*$|^65378(_65378)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65105(_65105)*$|^65296(_65296)*$|^65211(_65211)*$|^65317(_65317)*$|^65115(_65115)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65371(_65371)*$|^65214(_65214)*$|^65325(_65325)*$|^65354(_65354)*$|^65384(_65384)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65220(_65220)*$|^65277(_65277)*$|^65219(_65219)*$|^65213(_65213)*$|^65336(_65336)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65249(_65249)*$|^65112(_65112)*$|^65314(_65314)*$|^65385(_65385)*$|^65152(_65152)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65196(_65196)*$|^65252(_65252)*$|^65162(_65162)*$|^65271(_65271)*$|^65357(_65357)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65317(_65317)*$|^65360(_65360)*$|^65198(_65198)*$|^65256(_65256)*$|^65246(_65246)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65356(_65356)*$|^65359(_65359)*$|^65302(_65302)*$|^65118(_65118)*$|^65346(_65346)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65225(_65225)*$|^65307(_65307)*$|^65313(_65313)*$|^65189(_65189)*$|^65288(_65288)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65381(_65381)*$|^65292(_65292)*$|^65145(_65145)*$|^65325(_65325)*$|^65361(_65361)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65156(_65156)*$|^65184(_65184)*$|^65367(_65367)*$|^65302(_65302)*$|^65290(_65290)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65351(_65351)*$|^65116(_65116)*$|^65341(_65341)*$|^65123(_65123)*$|^65258(_65258)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65397(_65397)*$|^65302(_65302)*$|^65188(_65188)*$|^65187(_65187)*$|^65358(_65358)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65217(_65217)*$|^65107(_65107)*$|^65203(_65203)*$|^65377(_65377)*$|^65381(_65381)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65219(_65219)*$|^65308(_65308)*$|^65364(_65364)*$|^65277(_65277)*$|^65396(_65396)*$') then
    set med 80
  endif
  if as-path in (ios-regex '^65281(_65281)*$|^65336(_65336)*$|^65238(_65238)*$|^65381(_65381)*$|^65103(_65103)*$') then
    set med 80
  else
    set med 90
  endif
end-policy
</code>
</pre>
</div>

These outputs were collected after the full Internet BGP table scale is received and propagated further.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-in-dflt default-IPv4-Uni-10.0.54.6 policy profile
Policy profiling data
Policy : INGRESS-ROUTE-POLICY
Pass : 720100
Drop : 0
# of executions : 720100
<mark>Total execution time : 30119msec  !!!! about 30 seconds to process ingress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720100       4434msec  if as-path aspath-match ... then
                                       &lt;truePath&gt;
PXL_0_2           361          0msec    set local-preference 150
PXL_0_3        720100       3039msec    if as-path aspath-match ... then
                                         &lt;truePath&gt;
--- removed lines ---
GOTO :                                   PXL_0_3
                                        &lt;/reference&gt;
                                       &lt;/falsePath&gt;


RP/0/RSP1/CPU0:XR1#show pcl protocol bgp speaker-0 neighbor-out-dflt default-IPv4-Uni-UpdGrp-0.3-Out policy profile
Policy profiling data
Policy : EGRESS-ROUTE-POLICY
Pass : 720110
Drop : 0
# of executions : 720110
<mark>Total execution time : 106566msec  !!!! about 1.8 minutes to process egress updates</mark>

Node Id   Num visited      Exec time  Policy engine operation
--------------------------------------------------------------------------------
PXL_0_1        720110       2958msec  if as-path aspath-match ... then
                                       &lt;truePath&gt;
PXL_0_2             0          0msec    set med 70
PXL_0_3        720110       5222msec    if as-path aspath-match ... then
                                         &lt;truePath&gt;
PXL_0_4             3          0msec      set med 80
PXL_0_5        720110       4979msec      if as-path aspath-match ... then
                                           &lt;truePath&gt;
PXL_0_6                                     set med 80
--- removed lines ---
GOTO :                                   PXL_0_3
                                        &lt;/reference&gt;
                                       &lt;/falsePath&gt;
RP/0/RSP1/CPU0:XR1#
</code>
</pre>
</div>

# Conclusion

The performance of a route policy can be improved with native as-path match or collapsed regular expression patterns.

# Acknowledgements

I would like to thank Serge Krier for the original creation of this article.
