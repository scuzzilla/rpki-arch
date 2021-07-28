RPKI => Resource Public Key Infrastructure

IRR => http://www.irr.net/ --> Internet Routing Registry [The Internet Routing Registry (IRR) is a distributed set of databases allowing network operators to describe and query for routing intent]

ROA => Route Origin Authorization --> Document that authorize a certain provider to announce a specific prefix:
                                      it's typically signed by the provider (for example Swisscom) & the Authenticity is Certified
                                      by an independent organization like ARIN

ROV => Route Origin Validation [Valid, Invalid, NotFound]

VRP => Validate ROA Payload - Once a ROA is validated, the resulting object contains an IP prefix, a maximum length, and an origin AS number



Internet Hierarchy from the organizational point-of-view:

IANA =>  Internet Assigned Numbers Authority (allocates public Internet address space to Regional Internet Registries)
RIRs =>  Regional Internet Registries [AFRINIC-Africa, APNIC-Asia, ARIN-NAmerica, LACNIC-SAmerica, RIPE-Europe-CAsia-MEast]
LIRs =>  Local Internet Registries (Authorized by RIRs)


---

### Infrastructure Setup.

1. Krill - Child CA:
```
data_dir = "/var/lib/krill/data/"
log_type = "syslog"
admin_token = "e1bb6e95c21740f83dba1adb1ff19ade"
ip = "192.168.122.253"
port = 3000
```
