# Migration_from_L3VPN_to_EVPN_Type5
## Abstract
Communication Service Providers (CSPs) have successfully implemented advanced deployments of L3VPN (RFC 4364), L2VPNs (RFC 4664), LDP-based pseudowires (RFC 4447), BGP-based VPLS (RFC 4761), and LDP-based VPLS (RFC 4762) to provide Layer 2 and Layer 3 connectivity for their customers. If you're overwhelmed by the various standards and RFCs required to establish Layer 2 and Layer 3 connectivity across WAN backbone networks, you're not alone—I often feel the same when navigating the extensive standards for Layer 2 connectivity over WANs. To simplify these complexities, EVPN offers a straightforward solution. By utilizing BGP, EVPN streamlines Layer 2 and Layer 3 VPN services over an MPLS or IP backbone, eliminating the need for multiple protocols. This convergence of L2 and L3 services into a single solution makes configuration easier compared to older standards like LDP-based VPLS or BGP/MPLS L3VPN.

## Scope
In this article, we will explore Junos' implementation of EVPN Type 5 routes to extend Layer 3 connectivity across WAN backbone networks. It is assumed that the reader is familiar with Junos syntax and has a solid understanding of WAN backbone protocols

## Current Layer-3 VPN Configuration
The current Layer 3 VPN configuration from one of the Junos devices is shown below, and it's relatively straightforward to understand. A VRF named "test" has been created, which includes BGP connectivity to the CE through the BGP group "CE1." The VRF also applies the import and export policies "TEST_IMPORT" and "TEST_EXPORT" for routing control.
```
 test {
    instance-type vrf;
    protocols {
        bgp {
            group CE1 {
                type external;
                peer-as 65100;
                bfd-liveness-detection {
                    minimum-interval 1000;
                    multiplier 3;
                }
                neighbor 100.100.100.0;
            }
        }
    }
    route-distinguisher 172.172.172.1:100;
    vrf-import TEST_IMPORT;
    vrf-export TES_EXPORT;
}
```
### EVPN Type5 Routes Configuration 
Appended block of configuration provides funcationaliity to re-dsitrbute IP-VPN routes via MP-iBGP evpn signalling. 
```
vrf_sriov_1001 {
    instance-type vrf;
    protocols {
       
        bgp {
            group CE1 {
                type external;
                export evpn-to-ce;
                peer-as 65100;
                bfd-liveness-detection {
                    minimum-interval 1000;
                    multiplier 3;
                }
                neighbor 100.100.100.0;
            }
        }
        evpn {
            ip-prefix-routes {
                advertise direct-nexthop;
                encapsulation mpls;     
                import net-public-evpn-import;
                export net-public-evpn-export;
            }
        }
    }
    interface et-0/0/0.100;
    route-distinguisher 172.172.172.1:1001;
    vrf-import sriov_import;
    vrf-export sriov_export;
}


```
### Importing and Exporting IP Prefixes as EVPN Type 5 Routes
Routes learned from the CE (IP prefixes) via the eBGP group "CE1" will be converted into EVPN Type 5 routes using the routing policy "net-public-evpn-export." Subsequently, these routes will be advertised to remote PEs through MP-iBGP (EVPN signaling), with the vrf-export policy "sriov_export" controlling the advertisement of these EVPN Type 5 routes. On the remote PE, the vrf-import policy "sriov_import" will determine which IP-VPN prefixes are accepted from the advertising PE. Additionally, the EVPN import policy "net-public-evpn-export" will dictate which IP-VPN prefixes will be installed as EVPN Type 5 routes on the receiving PE. Once the EVPN Type 5 prefixes are installed on the receiving PE, they will be further redistributed to the CE via the policy "evpn-to-ce."

```    
policy-options {
    policy-statement evpn-to-ce {
        term 1 {
            from protocol evpn;
            then accept;
        }
    }
    
    policy-statement net-public-evpn-export {
        term bgp {
            from protocol bgp;
             then accept;
        }
        term evpn {
            from protocol evpn;
            then accept;
        }
        then accepts;
    }
    policy-statement net-public-evpn-import {
        then accept;
    }
    policy-statement  sriov_export {
        term 1 {
            from rib vrf_sriov_1001.inet.0;
            then reject;
        }
        term 2 {
            from rib vrf_sriov_1001.inet6.0;
            then reject;
        }
        term 3 {
            then {
                community add sriov;
                accept;
            }
        }
    }
    policy-statement sriov_import {
        term 1 {
            from community sriov;
            then accept;
        }
        term 2 {
            then reject;
        }
    }

 community sriov members target:65000:1001;
}
```

### Operational Verification from Left-Side PE
```
show bgp summary group CE1 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
100.100.100.0         65100      33202      33115       0       1 1w3d 12:04:10 Establ
  vrf_sriov_1001.inet.0: 3/3/3/0


show route receive-protocol bgp 100.100.100.0 
vrf_sriov_1001.inet.0: 8 destinations, 12 routes (8 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 100.100.247.0/24        100.100.100.0                           65100 I
* 100.100.248.0/23        100.100.100.0                           65100 I
* 100.100.254.0/24        100.100.100.0                           65100 I


show bgp summary group mpls-bb 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...

172.172.172.9         65000      22830      22664       0       1 1w0d 4:55:33 Establ
  bgp.evpn.0: 1/1/1/0
  vrf_sriov_1001.evpn.0: 1/1/1/0
  bgp.l3vpn.0: 0/0/0/0
  bgp.invpnflow.0: 0/0/0/0



show route advertising-protocol bgp 172.172.172.9 table vrf_sriov_1001.evpn.0 

vrf_sriov_1001.evpn.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  5:172.172.172.1:1001::0::100.100.248.0::23/248                   
*                         Self                         100        65100 I
  5:172.172.172.1:1001::0::100.100.247.0::24/248                   
*                         Self                         100        65100 I
  5:172.172.172.1:1001::0::100.100.254.0::24/248                   
*                         Self                         100        65100 I
```

#### Operational Verification from Right-Side PE

```
show bgp summary group mpls-bb
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.172.172.1         65000      22671      22834       0       0 1w0d 4:58:24 Establ
  bgp.l3vpn.0: 0/0/0/0
  bgp.invpnflow.0: 1/1/1/0
  bgp.evpn.0: 5/5/5/0
  vrf_sriov_1001.inetflow.0: 1/1/1/0
  vrf_sriov_1001.evpn.0: 5/5/5/0

 show route receive-protocol bgp 172.172.172.1 table vrf_sriov_1001.evpn.0 

vrf_sriov_1001.evpn.0: 8 destinations, 10 routes (8 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  5:172.172.172.1:1001::0::100.100.248.0::23/248                   
*                         172.172.172.1                100        65100 I
  5:172.172.172.1:1001::0::100.100.247.0::24/248                   
*                         172.172.172.1                100        65100 I
  5:172.172.172.1:1001::0::100.100.254.0::24/248                   
*                         172.172.172.1                100        65100 I



 show route table vrf_sriov_1001.evpn.0                        

vrf_sriov_1001.evpn.0: 8 destinations, 10 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:172.172.172.1:1001::0::100.100.248.0::23/248               
                   *[BGP/170] 1w0d 05:01:44, localpref 200, from 172.172.172.1
                      AS path: 65100 I, validation-state: unverified
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28
5:172.172.172.1:1001::0::100.100.247.0::24/248               
                   *[BGP/170] 00:07:50, localpref 200, from 172.172.172.1
                      AS path: 65100 I, validation-state: unverified
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28
5:172.172.172.1:1001::0::100.100.254.0::24/248               
                   *[BGP/170] 00:07:50, localpref 200, from 172.172.172.1
                      AS path: 65100 I, validation-state: unverified
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28
5:172.172.172.2:100::0::100.100.248.0::23/248               
                   *[BGP/170] 1w0d 05:01:44, localpref 100, from 172.172.172.2
                      AS path: 65100 I, validation-state: unverified
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE1-to-left-side-PE2-1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.24
5:172.172.172.9:1001::0::100.100.253.0::24/248               
                   *[EVPN/170] 1w2d 04:49:29
                       Fictitious

                       

show route table vrf_sriov_1001.inet.0 

vrf_sriov_1001.inet.0: 6 destinations, 10 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

100.100.100.4/31   *[Direct/0] 1w2d 04:47:45
                    >  via et-0/0/0.100
100.100.100.5/32   *[Local/0] 1w2d 04:47:45
                       Local via et-0/0/0.100
100.100.247.0/24   *[EVPN/170] 00:05:53
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28
100.100.248.0/23   *[EVPN/170] 1w0d 04:59:48
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28
                    [EVPN/170] 1w0d 04:59:48
100.100.254.0/24   *[EVPN/170] 00:05:53
                    >  to 10.0.1.38 via et-0/0/4.0, label-switched-path right-side-PE2-to-left-side-PE1
                       to 10.0.1.32 via et-0/0/2.0, label-switched-path Bypass->10.0.1.38->10.0.1.28


show bgp summary group CE1 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
100.100.100.4         65110      29339      29179       0       0 1w2d 4:50:16 Establ
  vrf_sriov_1001.inet.0: 1/1/1/0


show route advertising-protocol bgp 100.100.100.4 
vrf_sriov_1001.inet.0: 6 destinations, 10 routes (6 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 100.100.247.0/24        Self                                    65100 I
* 100.100.248.0/23        Self                                    65100 I
* 100.100.254.0/24        Self                                    65100 I
```
