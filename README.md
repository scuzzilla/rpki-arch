---
title: RPKI (My)Lab Environment
tags: ['RPKI', 'rpki-client', 'nlnetlabs', 'BGP', 'OpenBSD']
status: draft
---

# RPKI (My)Lab Environment

## Context 

Recently I decided to setup a dedicated test Environment to improve my knowledge around *Resource Public Key Infrastructure* (RPKI).
Now that the testbed is finally up & running I would like to share my journey with the **"Networkers Community"**, eventually brainstorming together on how it could be tuned-up.


## Network Setup

Using [eve-ng](https://www.eve-ng.net/) I linked together few routers with the main aim of simulating a BGP IP Hijacking attempt and eventually try to neutralize this threat implementing RPKI.
For sake of simplicity I deployed only three routers, each one of them belonging to a different BGP Autonomous System. 

The router belonging to **ASN 30** was compromised and started announcing an IP address which overlaps with an IP prefix range *legitimately* advertised by **ASN 20**. From now on, 
whoever is behind **ASN 10** and will try to reach out to **170.0.0.1/32** will be forwarded to the rogue router belonging to **ASN 30**.

<p align="center">
    <img src="https://db3pap005files.storage.live.com/y4mY2Xf9N7gMtcL9S7vHg3a4GylwAxPyF0dBtxC55d8J0ob07Qav6D-xU0CESAD00N2AcJDjT-LIFPg5qYAfyZtGlLaRSYed_DuYnHCURcuvc3cqlBCYzTX2MRvobpI3Wn1xKmHzAf170ieBbiMRYtnNzly-U3QURpEJjXGxMANJvUm7GxuBKCfkUlbw15xf9f8?width=488&height=337&cropmode=none" width="488" height="337" alt="Network Setup"/>
</p>

The BGP protocol will simply prefer the more specific prefix (170.0.0.1/32) over the less specific one (170.0.0.1/24). The traffic will flow towards the compromised ASN 30 via next-hop 10.0.0.6 (instead of 10.0.0.2).

```c
RP/0/0/CPU0:router-asn10#sh bgp sessions
Mon Jul 26 14:28:00.527 CET

Neighbor        VRF                   Spk    AS   InQ  OutQ  NBRState     NSRState
10.0.0.2        default                 0    20     0     0  Established  None
10.0.0.6        default                 0    30     0     0  Established  None

RP/0/0/CPU0:router-asn10#show bgp ipv4 unicast neighbors 10.0.0.2 routes | b ^Status
Mon Jul 26 14:28:13.456 CET
Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
*> 170.0.0.0/24       10.0.0.2                 0             0 20 i

Processed 1 prefixes, 1 paths

RP/0/0/CPU0:router-asn10#show bgp ipv4 unicast neighbors 10.0.0.6 routes | b ^Status
Mon Jul 26 14:28:23.945 CET
Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
*> 170.0.0.1/32       10.0.0.6                 0             0 30 i

Processed 1 prefixes, 1 paths

RP/0/0/CPU0:router-asn10#traceroute 170.0.0.1
Mon Jul 26 14:28:48.354 CET

Type escape sequence to abort.
Tracing the route to 170.0.0.1

 1  10.0.0.6 9 msec  *  0 msec 
RP/0/0/CPU0:router-asn10#

```

## RPKI IT Infrastructure Setup

Whenever it comes to RPKI [nlnetlabs](https://www.nlnetlabs.nl/) is one of the main reference: they wrote extremely clear [Documentation](https://rpki.readthedocs.io/) and last but not least 
they developed some of they key software components I (also) utilized within (My)Lab Infrastructure:

- [Krill](https://krill.docs.nlnetlabs.nl/) can be classified as "Certificate Authority Software" and in my setup is mainly responsible for Forging and Publishing the *Route(s) Origin Authorization* (ROAs) to the Parent Certification Authority.

- [RTRTR](https://rtrtr.docs.nlnetlabs.nl/) can be classified as *RPKI-to-router protocol* (RTR) Server Software ([RFC 6810-v0](https://tools.ietf.org/html/rfc6810.html) & [RFC 8210-v1](https://tools.ietf.org/html/rfc8210.html)).
Its main job is to dispatch the validated ROAs to the BGP enabled routers.

Other than nlnetlabs, [OpenBSD's](https://www.openbsd.org) [rpki-client](https://www.rpki-client.org/) is with no doubt the core component of the whole solution. It's responsible for:

1. ROAs (X.509 Certificates) synchronization (via RSYNC or RRDP) from a given *Trust Anchor* (TA). The TA is usually a *Regional/Local Internet Registry* (RIR/LIR) organization.
2. Validating the chain of trust for the associated ROAs (including checking relevant Certificate Revocation Lists).
3. Caching a list of the *validated ROAs' Payloads* (VRPs) in various formats, for example JSON.

<p align="center">
	<img src="https://db3pap005files.storage.live.com/y4mTPMo0mwmOqtt1qwXrckrvnUU1joUjU8Pqz6aVwvg_hDf5olmyXaOgE9Ed2xGTPM_bfIBiEW9WD_xE3MZXJFkPP90mMOw028DO7aifFZsyn2rwwO0tnhSSZj6rOpIb90hhZxlmR62DWBZipUE5XpeLDxAzkdqGbjv9wvdUCL1B8U2Rnw76pDtsUXDzLX_GV09?width=493&height=716&cropmode=none" width="493" height="716" alt="RPKI IT Infrastructure Setup"/>
</p>


Since I'm working within a testing environment I decided to collapse all services on a single Linux host. However, potentially, nothing is preventing the ability to horizontally scale each one of the involved services.

For what concerning the software installation & configuration I recommend to refer to the respective official documentation. Nevertheless, for sake of completeness, I will include the main configuration files within the References & Resources section.

#### KRILL

Krill can be deployed using two different models: [Hosted RPKI](https://rpki.readthedocs.io/en/latest/rpki/implementation-models.html#hosted-rpki) or [Delegated RPKI](https://rpki.readthedocs.io/en/latest/rpki/implementation-models.html#delegated-rpki): I chose the latter.

After successfully completing both the [Repository setup](https://krill.docs.nlnetlabs.nl/en/stable/get-started.html#repository-setup) & the [Parent setup](https://krill.docs.nlnetlabs.nl/en/stable/get-started.html#parent-setup) I was ready to start Publishing new ROAs.

I named the Child CA "rpki-alfanetti" and as you can see from the text snippet below the relationship with the Parent CA (testbed offered by nlnetlabs) is in Status: **Success**.

The Parent CA is certifying that I'm entitled over some specific resources: **"asn: AS20, v4: 170.0.0.0/24"** is one of them. Third parties can, in a second stage, download the Signed Certificate (to be verified) which proves the ownership over that specific resource.
In compliance with the RPKI terminology, the resource I'm dealing with is named Route Origin Authorization (also known as **ROA**)   
```yaml
root@rpki01:~# krillc parents statuses --token e1bb6e95c21740f83dba1adb1ff19ade --ca rpki-alfanetti
Parent: testbed
URI: https://testbed.rpki.nlnetlabs.nl/rfc8181/rpki-alfanetti
Status: success
Last contacted: 2021-07-27T08:50:00+00:00
Next contact on or before: 2021-07-27T09:00:00+00:00
Resource Entitlements: asn: AS10, AS20, v4: 170.0.0.0/24, 172.0.0.0/24, v6: 
  resource class: 0
  issuing cert uri: rsync://testbed.rpki.nlnetlabs.nl/repo/ta/0/1040FA57F99F4B3DE2626F6EE1C56664CB81D2C8.cer
  received certificate(s):
    published at: rsync://testbed.rpki.nlnetlabs.nl/repo/testbed/0/660E433B340BC5B12B204F1544C6EA18A5931DB4.cer
    resources:    asn: AS10, AS20, v4: 170.0.0.0/24, 172.0.0.0/24, v6: 
    cert PEM:

-----BEGIN CERTIFICATE-----
MIIFsTCCBJ ... MojHUKkp30dIbbpo49FocyZyI58lFI7DsDVmXn9Bz0sAeYRB
SNviq7K+O4SS/RZezDuY/5MEXHvDWsZXldfX1r+RxsgXi0/5fuudLU4CYlWQGzs
-----END CERTIFICATE-----

```

#### OpenBSD's rpki-client

The OpenBSD's rpki-client is periodically syncing with the Parent CA looking for new ROAs. In order to do that is taking as input some information regarding the Trust Anchor (belonging the Parent CA) in particular there's the certificate used to validate the received ROAs.

The rpki-client is somehow hiding the complexity coming from the cryptography required in order to validate the received ROAs.

Once the validation process is completed the ROAs information are extracted from the associated certificates and presented in clear text format (for example JSON); ready to be "digested" by the routers.
```json
### root@rpki01:~# cat /var/lib/rpki-client/json
{
        "metadata": {
                "buildmachine": "rpki01",
                "buildtime": "2021-07-26T08:04:49Z",
                "elapsedtime": "133",
                "usertime": "0",
                "systemtime": "0",
                "roas": 6,
                "failedroas": 0,
                "invalidroas": 0,
                "certificates": 49,
                "failcertificates": 0,
                "invalidcertificates": 0,
                "tals": 1,
                "talfiles": "/etc/tals/testbed.tal",
                "manifests": 49,
                "failedmanifests": 43,
                "stalemanifests": 0,
                "crls": 6,
                "repositories": 9,
                "vrps": 6,
                "uniquevrps": 6
        },

        "roas": [
                { "asn": "AS65101", "prefix": "10.0.0.0/23", "maxLength": 24, "ta": "testbed" },
                { "asn": "AS20", "prefix": "170.0.0.0/24", "maxLength": 24, "ta": "testbed" },
                { "asn": "AS10", "prefix": "172.0.0.0/24", "maxLength": 24, "ta": "testbed" },
                { "asn": "AS65001", "prefix": "192.168.0.0/23", "maxLength": 24, "ta": "testbed" },
                { "asn": "AS37708", "prefix": "196.1.0.0/24", "maxLength": 24, "ta": "testbed" },
                { "asn": "AS65001", "prefix": "2001:db8::/32", "maxLength": 64, "ta": "testbed" }
        ]
}

```

#### HTTP Server/RTRTR & BGP Routers configuration

A basic HTTP Service (python3 -m http.server 8081) is allowing the RTR Server to access the JSON file storing the ROAs' records. RTRTR is now listening for incoming connections from the BGP routers on the TCP socket <192.168.122.253:8282>.

The RTR Server should be reachable by the involved routers and the below configuration should be present on each one of them. 
```c
router bgp <ASN>
 rpki server 192.168.122.253
  transport tcp port 8282
  refresh-time 30
 !
```

From the routers, checking the connectivity status to the RTR Server should display something similar to this:
```c
RP/0/0/CPU0:router-asn10#sh bgp rpki server summary      
Tue Jul 27 14:29:11.497 CET

Hostname/Address        Transport       State           Time            ROAs (IPv4/IPv6)
192.168.122.253         TCP:8282        ESTAB           04:41:46        5/1

```

#### Enabling BGP destinations validation

Now that all the legitimate ASNs are participating to the RPKI process I should be able to quickly identify the **Invalid** BGP destinations. 
As expected, displaying the IPv4 unicast BGP database of the router belonging to ASN 10 is reveling that the destination towards ASN 30 is not verified:
```c
RP/0/0/CPU0:router-asn10#show bgp ipv4 unicast origin-as validity | b ^Origin-AS
Tue Jul 27 14:42:25.998 CET
Origin-AS validation codes: V valid, I invalid, N not-found, D disabled
    Network            Next Hop            Metric LocPrf Weight Path
V*> 170.0.0.0/24       10.0.0.2                 0             0 20 i
I*> 170.0.0.1/32       10.0.0.6                 0             0 30 i
 *> 172.0.0.0/24       0.0.0.0                  0         32768 i

Processed 3 prefixes, 3 paths
``` 
However, by default on CISCO-XR routers, invalid destinations are not automatically discarded. I should manually enable the **valid** destination selection with this command:
```c
router bgp <ASN>
 bgp bestpath origin-as use validity
```
Immediately after the new configuration is committed only the **valid** destinations are selected and any risk of IP Hijacking is finally defeated:
```c
RP/0/0/CPU0:router-asn10#traceroute 170.0.0.1
Tue Jul 27 14:55:23.282 CET

Type escape sequence to abort.
Tracing the route to 170.0.0.1

 1  10.0.0.2 0 msec  *  9 msec

```

## References & Resources

- Clone & Test:
1. [RPKI (My)Lab @github](https://github.com/scuzzilla/rpki-arch)

- Documentation:
1. [nlnetlabs](https://www.nlnetlabs.nl/)
2. [krill](https://krill.docs.nlnetlabs.nl/en/stable/index.html)
3. [OpenBSD rpki-client](https://man.openbsd.org/rpki-client)
4. [RTRTR](https://rtrtr.docs.nlnetlabs.nl/)
5. [EVE-NG](https://www.eve-ng.net/)

- NET Resources:
1. [router ASN 10 configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/net/router1.cfg)
2. [router ASN 20 configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/net/router2.cfg)
3. [router ASN 30 configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/net/router3.cfg)
4. [net verification commands](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/net/network-cmds.cfg)

- Service Resources:
1. [Krill configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/services/krill.conf)
2. [rpki-client systemd service configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/services/rpki-client.service)
3. [RTRTR configuration](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/services/rtrtr.conf)
4. [ROAs JSON format](https://raw.githubusercontent.com/scuzzilla/rpki-arch/main/rpki-lab/services/ROAs.json)