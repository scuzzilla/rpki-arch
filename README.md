# RPKI (My)Lab Environment

## Context 

Recently I decided to setup a dedicated test Environment to improve my knowledge around Resource Public Key Infrastructure (aka RPKI).
Now that the *testbed* is finally up & running I would like to share my journey with the Community, eventually brainstorming together on how it could be tuned-up.


## The use-case

Using [eve-ng](www.eve-ng) I linked together few routers with the main aim of simulating a BGP IP Hijacking & to try to overcome the security issue implementing RPKI.
For sake of simplicity I deployed only three routers, each one of them belonging to a different BGP Autonomous System. 

The router belonging to **ASN 30** was compromised and started announcing an IP address which overlaps with an IP prefix range *legitimately* advertised by **ASN 20**. From now on, 
whoever is behind **ASN 10** and will try to reach out to **170.0.0.1/32** will be forwarded to the compromised router behind AS 20.

<p align="center">
    <img src="rpki-lab/net/rpki_network.jpg">
</p>