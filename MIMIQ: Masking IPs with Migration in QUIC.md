# Overview
QUIC offers the capability to migrate connections to other hosts (IPs). This
allows new technologies such as MIMIQ to mask source IPs by automatically
migrating to new IPs.

Existing Address Hiding Protocols (AHPs) are coarse grained, and can be executed
between flows. This does not prevent identification from single flows.

QUIC's IP migration technology is designed for migrating mobile clients as they
roam. Leveraging this, clients can use multiple IPs on the same router to
obfuscate individual flow analysis.

Requires an IP allocation service.

There is an IP allocation service, called the *trusted provider* that all
clients on a network share. There may be colluding clients as well. Client
connects to a server (using IPSec as well) and attackers can see which server
they are connecting to but not the data.

# Design
The IP allocation service has a global view  of the IP space and can modify
switches to direct packets as necessary.

QUIC allows for encryption (using TLS cert), thus registering and changing
*connection IDs* is obscured.

# Questions
1. How significant is it to hide IP identity within a network?
2. Seems like Statistical Disclosure Attacks would be effective here.
3. This seems like it just puts a middle man in discovery process. For example,
   it doesn't seem useful in a home situation. If I have IP 10.0.0.0 or 10.0.0.1
   is insignificant, both are my router. Similarly at a university switching
   from 10.0.0.0 to 10.127.0.0 would both be flagged as University comp.