# BGP 01 - Intro & Convergence

#### Resources:
You should read and understand:

**Paper:** _[Stable Internet routing without global coordination](https://www.cs.princeton.edu/~jrex/papers/sigmetrics00.long.pdf)_, Lixin Gao & Jennifer Rexford (GR'01)

**BGP Path Selection**: https://learn.nsrc.org/bgp/path_selection


# Overview of BGP

The Border Gateway Protocol (BGP) is a protocol for routers to exchange information about reachability. It is how participants on the Internet determine where to route traffic.

BGP is distributed and leaderless. There is no holistic route table; each participant operates autonomously and locally.

## Basics

* Each distinct "entity" on the Internet is an Autonomous System, or **AS**
* BGP is used both *internally* to an AS, as well as *externally* between ASs
* Routers use BGP to exchange routes
* "Bellman-Ford" inspired algorithm (except for all the ways it's different)
* When a router's routing table changes, it notifies all peers of that change.
* When a router receives a new route, it recomputes its routing table and, if changed, updates all peers.

### Routes

BGP routes exchange "route update" messages. Each update message announces or withdraws one or more paths.

Each path has a prefix (e.g. 7.4.0.0/16) as well as a number of attributes Some attributes are transitive, some are propagated within the IGP, and some are local to the peering. There are also pseudo-attributes applied locally by the routing process that are never part of the protocol

Some interesting attributes:
* **AS Path:** (transitive) the path to the originating AS
* **Next-Hop:** (intra-AS) For eBGP, (usually) the IP of the originating router. Can be different for route reflectors.
* **Multi-Exit Discriminator (MED):** (local) Used when two ASs have multiple locations
* **Community:** (intra-AS) Indicates that the route should have special behavior
* **Weight:** (local) An arbitrary, local-only tiebreaker
* **Local Preference:** (intra-as) An arbitrary, intra-AS tiebreaker


## Example 

The classic textbook example. We'll later see where this can go wrong.

![basic BGP topology](https://blog.catchpoint.com/wp-content/uploads/2019/10/bgp-routing-diagram.png)

## Path selection

Every BGP speaker is individually responsible for selecting between competing paths to a prefix. Path selection uses the following algorithm to "break ties"

0. Prefix length - more specific always wins
1. Ignore path if no route to next hop
2. (historical) Do not consider IBGP if not synchronized
3. Highest weight (local to router)
4. Highest local preference (intra-as)
5. Prefer locally-originated route
6. **Shortest AS-path**
7. (historical) Lowest "origin code"
8. Lowest MED
9. Prefer EBGP over IBGP
10. Lower IGP metric
11. Things get weird:
    - if multipath is enabled, install parallel paths. Finished.
    - (non-standard, but widely adopted) if router-id is not the same, select oldest path
12. Lowest router-id (arbitrary tiebreaker)
13. Shortest cluster-list (this is a route-reflector thing)
14. Lowest neighbor address (again, arbitrary tiebreaker)

# Convergence problems

In the simple "path-vector" algorithm, BGP always reaches a stable state. However, there are configurations for which BGP does not quiesce. The term for this is _oscillating routes_.

BGP can reach this state because AS-PATH length is 7th on the list of path-selection factors. ISPs may choose longer paths for a number of reasons, all relating to internal policy <sup>money</sup>.

It has been proposed that ISPs can upload their routing polices to a centralized registry (the _IRR_), which can run analysis to detect potential problems. This is, however, impractical for engineering, political, and algorithmic reasons - such analysis is NP-complete! 


## Example potential problem:

![three-node BGP topology](https://i.imgur.com/LzvDpcs.png)

Consider a route to prefix `d` originated by network `AS0`. If for whatever reason, `AS1` and `AS2` prefer the routes from each other, the system may enter a state where routes oscillate. To understand how this may happen, one must consider the _activation sequence_ of BGP speakers, for which the paper formalizes.

In the paper, an _activation_ of a given speaker _v_ (for vertex) is defined as:

1. All of the peers of the speaker transmit all of their paths, save any filted by _export policies_, to the speaker _v_.
2. _v_ applies any _import policies_ to the received routes
3. _v_ executes its _path selection_ algorithm.

So what is the sequence of events that can cause route oscillations?

0. AS1 and AS2 both select the direct route to _d_ via AS0
1. They both announce _d_ to each other. AS1 announces as-path `AS1 AS0`. AS2 announces `AS2 AS0`.
2. They both execute route selection. AS1 prefers path `AS1 AS2 AS0`, and AS2 prefers path `AS2 AS1 AS0`. AS0 is now unreachable.
3. AS1 activates. AS1 announces as-path `AS1 AS2 AS0` to AS2. Loop-detection rejects this route, so AS2 chooses path `AS2 AS0`

So, this configuration of routes and policies is not stable for all possible activation sequences.

# Safe BGP Policy Set

GR'01 propose a _safe subset_ of possible policies that ASs can apply. They later show that this subset is guaranteed to safely converge. This is possible because the Internet is not an "arbitrary mesh". Rather, its topology reflects certain commercial structures. 

By defining restrictions on import and export policies in line with this larger structure, it is possible to ensure that all possible toplogies and activation timelines converge.

## Hierarchical AS-graph

Commercial realties mean that the AS graph is effectively hierarchical. Let's look at why this is the case.

### BGP session types

The paper breaks down BGP sessions in to two types: Customer-Provider and Peer-Peer. The relationships are defined by the routes that are exported.

* **Customer -> Provider** Customers export their own routes, as well as routes of their customers, to their providers. They do not export routes learned from peers.
* **Provider -> Customer** Providers export all routes to customers.
* **Peer -> Peer** Peers export their own routes, as well as routes of their customers.

**Why is this the case?**

If a customer exports peer routes to providers, they would be providing peers free transit.
If a peer exports provider or peer routes to peers, they would be providing peers free transit.

### Provider Hierarchy

It can be assumed that the customer -> provider relationship is a DAG. That is to say, if A pays B for transit, and B pays C for transit, then C will not pay A for transit.

This follows from the basic reasoning that smaller providers purchase transit from larger, more widely dispersed providers.

**Note:** In the 20 years since this paper was written, the presence of hyperscalers (e.g. AWS) that are simultaneously transit customers as well as wide-reach networks are a curious exception to this assumption.

## Policy guidelines

The paper proposes several reasonable policy guidelines that have the effect of guaranteeing stable routing.

### Guideline A

**An AS will always prefer a customer route, regardless of AS-path length, over that from a peer or provider.**

This guideline assumes that any two ASs could be peers.

**Proof Sketch:** When a destination _d_ is originated by `AS0`, it propagates in a partial ordering based on the customer -> provider DAG (along with direct peers of `AS0`). Since the partial ordering also refelects the AS path length, any subsequent routes received by an arbitrary AS won't be selected.


### Guideline B

**An AS will always prefer a customer and peer route equally over that from a provider.**

It is possible to relax these guidelines somewhat and allow peers and customers to be preferred equally, by expanding the assumption of hierarchy.

Specifically, the paper assumes that the AS connectivity graph is still a DAG, even when peering relationships are taken in to account. 

However, the preference must be *equal*. Peers cannot be preferred over customers.

