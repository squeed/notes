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


## Example 

