# Overview
Previous Deep Packet Inspection (DPI) evasion techniques involve labor intensive
techniques to manually craft packets that slip through. 

Given that existing DPI middleboxes are stateful machines, not necessarily
dissimilar to endhosts, this team analyzes end hosts to systematically determine
where and why packets are dropped or ignored.

Using this method the team is able to generate 10k adversarial packets in
seconds which are effective at circumventing test middleboxes

# Introduction
DPI is effective in many scenarios.

To be effective it must keep track of TCP state to reconstruct the payload.

TCP spec is written in English (natural language) and is inherently ambiguous.
This leads to vendor specific implementations. For simplification DPI
middleboxes also implement their own simplified state machines.

Due to discrepancies between DPI middleboxes and endpoints we have two special
types of packets: *insertion* and *evasion*.

**Insertion**: Packet that is accepted and acted upon by middlebox, changing its
state, which the endhost will ignore

**Evasion**: Packet which is ignored by DPI box but accepted and acted upon by
endhost

Via insertion and evasion packets the DPI middlebox and endhost are in different
states, meaning they no longer construct the same payload, causing the DPI
middlebox to fail to catch any sensitive info.

Previous research requires manual creationg of insertion and evasion packets.
This is manually intensive. Enumeration is also difficult there are $2^{160}$
possible 20 byte TCP headers.

TCP implentations of DPI middleboxes are obscure and hard to obtain, TCP enhosts
are readily available.

Process:
1. Analyze endpoints using symbolic execution to determine which packets are
   ignored and which are accepted.
2. Pass packets through a DPI middlebox to determine whether middlebox can still
   function correctly as a DPI, or if its state is incorrect.

# Threat Model
Single DPI device between client (user) and server (endhost). DPI device can
read all packets exchanged between them. TCP only.

Assumes DPI device has its own TCP implementation, and is deterministic. And
inspection will lead to observable effects, blocking or resetting a connection.

DPI implementation is blackbox.

Assume only client will craft messages to fool DPI middlebox.

Assume endhost uses a public TCP implementation (i.e., Linux) which can be used
as a whitebox. As such this assumes the server is not colluding with the client,
if so other techniques are available.

## Definitions
* Drop: A packet is *dropped* if it doesn't change the state of the machine and
  doesn't generate output. This appears to be akin to ignored.
* Accept: A packet is *accepted* if the state machine changes state or the
  output is non-empty due to the packet.
* Syncronized: A DPI middlebox and TCP endhost are synchronized if their output
  on a sequence of packets is identical.
* Bad keyword: Any content that will trigger an alarm from a DPI middlebox.
* Evasion Packet: In a sequence of packets $P_1,...,P_n$, $P_n$ is an evasion
  packet if:

    1. The server will *accept* $P_1,...,P_n$.
    2. The server and middlebox are *synchronized* for $P_1,...,P_{n-1}$.
    3. The server and middlebox are *de-synchornized* for $P_n$ as the middle
        box will *drop* $P_n$ and thus fail to output the payload of $P_n$ or
        future packets.

* Insertion Packet: In a sequence of packets $P_1,...,P_n$, $P_n$ is an
  insertion packet if:

    1. The server will *accept* $P_1,...,P_{n-1}$, but *drop* $P_n$.
    2. While handling $P_1,...,P_{n-1}$ the server and middlebox are
       *synchronized*.
    3. $P_n$ will *de-synchronize* the server and middlebox as the middlebox
       will *accept* $P_n$.

# Workflow of SynTCP
There are two stages of SynTCP:
1. Offline phase
2. Online phase

## Offline phase
Run concolic execution on server TCP implementation to find all accept/drop
states and record them in the form of symbolic symbols.

## Online phase
Turn the symbolic symbols into concrete packets that force a TCP server into a
given state and test whether the DPI middlebox behaves differently:
* If there is a suspected Insertion packet, send that packet sequence, assuming
  last packet is dropped by server, then send a Bad Keyword packet and determine
  if the DPI box state has changed enough to ignore it.
* If there is a suspected Evasion packet, send that packet sequence, assuming
  the middlebox is behind in state. Send a Bad Keyword packet and determine if
  so. 

Either way always send Bad Keyword packets afterwards to determine if DPI
middlebox is in incorect state.

# Questions
* Sounds a lot like an earlier paper we read on using ML to generate packets to
  fool censorship, combine work?
* Process step 1 seems good for finding potential insertion packets, but
  unhelpful for finding evasion packets?
* Presumably DPI middleboxes fail open? In the `rst` example, why would the
  middlebox not just block all communication to endhost because of reset
  connection? Or would it assume next packet is starting a new connection?
  Surely the box doesn't behave as, I think this connection is closed, so even
  though there is communication I'm going to ignore it?
* My understanding of the state machine is that it acts on headers to determine
  what to do with the payload? Or is that incorrect?
