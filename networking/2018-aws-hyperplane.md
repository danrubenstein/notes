### Load Balancing At Hyperscale - AWS HyperPlane

* [Video](https://atscaleconference.com/videos/networking-scale-2018-load-balancing-at-hyperscale/), [Source](https://medium.com/@copyconstruct/best-of-2018-in-tech-talks-2970eb3097af)
* Speakers:  Alan Halachmi, Colm MacCarthaigh

#### HyperPlane, what is it?

Layer 4 load balancer that sits on top of a lot of AWS' managed
services (like S3 or EFS).

HyperPlane is a cellular and zonal service. AWS' global footprint is built on
regions (e.g., US-East-1 (Virginia)), those regions are built of Availability
Zones (US-East-1e), and those regions are built of cells.

HyperPlane is applied on a per cell level, with multiple nodes in each cell to
ensure resiliency.

Each node consists of:
- Top: manages IPv4 SRC -> DST packet transmission connections
- Flow Manager: holds the state of these connections
- Decider: picks what resource (EC2, Lambda, etc.).

Once the decider has picked a mapping from the outside to the inside, that
is preserved until the connection is closed from either side, or in the case of
failover.

Fun fact: _HyperPlane is run on EC2s, because everything is run on EC2s._

#### System Reliability

How does AWS ensure that customers experience functional single-tenancy without
any of the problems of noisy neighbors?

SHOCK principle (Self-Healing and Constant Work): a system should always be in
the repair state, and should not have to do additional work when something
breaks. This can lead to thundering herd or other similar problems.

HyperPlane: when a node breaks down, AWS lets the connection die, and relies
on typical TCP retry mechanisms on the client side to enable the data to get
through, rather than doing additional retry work or amplification.

Shuffle Sharding: In each HyperPlane cell, a single cloud customer is put in
some number of nodes n, which compared to the total number of HyperPlane nodes
in an cell, means that the probability of a nosiy neighbor problem approaches
zero (combinatorial expansion is good!).
