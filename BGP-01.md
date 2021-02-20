# BGP 01 - Intro & Convergence

#### Resources:
You should read and understand:

**Paper:** _[Stable Internet routing without global coordination](https://www.cs.princeton.edu/~jrex/papers/sigmetrics00.long.pdf)_, Lixin Gao & Jennifer Rexford

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
    - if multipath is enabled, install parallel paths
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
2. 