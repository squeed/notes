# BGP 03: Security & Attacks

#### Links

Paper: [SICO: Surgical Interception Attacks by Manipulating BGP Communities](https://www.cs.princeton.edu/~jrex/papers/sico19.pdf)

RPKI: [NLNet RPKI Docs](https://rpki.readthedocs.io/en/latest/index.html)

## Background

BGP misuse has been studied for some time. Attackers can attract traffic destined for a victim AS relatively easily. Why does this work?

- BGP has no security inherent in the protocol
- All announcements are, by default, trusted.
- Prefix length overrides **all other attributes**

BGP hijacks can be divided in to two basic categories:

- *Hijacking*, where traffic is simply claimed
- *Interception*, where a prefix is announced, examined in some way, and *forwarded to the victim*.


Interception is clearly more powerful than hijacking.
- Enables traffic analysis
- Allows for sniffing of full transactions
- May not be noticed by the victim.

However, it is also more difficult. The complexity is in maintaining a route back to the victim that is not also affected by the malicious announcements.


### Basic Hijack Attack

Depressingly simple.

If the target is announcing, say, `10.0.0.0/20`, and you'd like to claim all traffic destined for those prefixes, you simply announce `10.0.0.0/21` and `10.0.8.0/21` and you win.

Now you can do exciting things like

1. Get a [certificate from LetsEncrypt](https://petsymposium.org/2017/papers/hotpets/bgp-bogus-tls.pdf)
2. Double-spend some [bitcoin](https://btc-hijack.ethz.ch/)
3. Poison some [DNS entries](https://www.internetsociety.org/blog/2018/04/amazons-route-53-bgp-hijack/) to steal Ethereum
4. Or just take down some websites (too numerous to link)

### Hijack countermeasures

(Worth a quick discussion)

#### Prefix Filters

ISPs statically configure all IPs a customer is ever allowed to announce. Out-of-band.

Only works for "leaf nodes" in the tree. Doesn't work when an ISP does something wrong.

#### RPKI

Automated, signed version of the above.

#### Hijack detection

Some cool tools that use heuristics to detect potential hijacks. See [ARTEMIS](https://www.inspire.edu.gr/wp-content/pdfs/artemis_TON2018.pdf).

Upon detecting a hijack, either de-aggregate to a longer prefix, or use a third-party vendor to flood legitimate announcements (and tunnel traffic back).


## BGP Interception

BGP interception combines a partial *BGP hijack* with various tricks to **maintain connectivity to the victim**. This enables, then, remote MITM attacks.

One option is to tunnel traffic to a separate exit point very close to the victim, but this is inflexible and requires an ISP that allows *spoofing*, which is hard to find. Thus, tunneling is not commonly used.

The other option is to maintain a return path, and only hijack traffic for a portion of the internet. The challenge is how to **prevent** the hijacked route from being accepted by the chosen return path. This is known as **announcement shaping**.

There are several announcement-shaping techniques.

### Announcement-shaping

In order for this approach to work, the attacker needs

- Diverse connectivity (a.k.a. Multi-homing)
- Providers without RPKI
- A deep understanding of the paths back to the victim

The goal is to craft special BGP announcements such that a statically-chosen path to the victim will ignore the bogus route, while as many others as possible will propagate it.

### Shaping via Loop Prevention (Path Poisoning)

This is classic announcement-shaping as first demonstrated by Pilosov & Kapela when they hijacked most of Defcon 16's incoming traffic in 2008. It exploits the fact that routers will reject announcements that contain their own AS.

Assume the victim is at `10.0.0.0/20`.


![](https://i.imgur.com/6DRs67b.png)

In this example, AS100 announces the route `10.0.0.0/24 10 20 200 100`.

The fact that is a longer prefix means that all other ASs prefer it, except for the route back to the victim.

#### Current effectiveness?

This attack is not nearly as effective anymore:

- BGP Hijack detection, e.g. ARTEMIS, will notice the deaggregation quickly
- Peer locking + AS-path filtering, now won't let you announce a Tier-1's AS

So it is useful to find an attack that doesn't rely on deaggregation.

### Shaping via business relationships

In the Gao-Rexford model, ASs will always prefer routes learned from a customer to that of a peer, and a peer to that of a provider (a.k.a. follow the money). One can exploit this effect to maintain a path back to the vitcim.

### Shaping via communities

Rather than relying on second-order effects, this attack uses *BGP Communities* to block route propagation at key points. This is the focus of this particular paper.

## BGP Communities

BGP communities are additional attributes attached to an announcement that may affect route decision.

These attributes are *partially transitive* - that is to say, some of them may cross AS boundaries.  As a rule, communities only flow from customer to provider - never from peer to peer. This means that a community attached to an announcement stops at the Tier-1.

### Common communities and their effects

Every ISP implements communities differently (or not at all). But most ISPs provide one or more of the following:

- `LowerPref` - Lower the preference of this route to that of a peer
- `NoExportSelect(X)` - Don't propagate this announcement to AS `X`
- `NoExportAll` - Don't propagate this announcement to all providers and peers. Almost universally available.


## SICO: Surgical Attacks using Communities

They key insight from the paper is that careful use of communities can enhance BGP attacking capability. Specifically, it can make it easier to gather more traffic, because more ASs can learn the "bogus" route while preserving connectivity.

### Notation and setup

The rest of the paper assumes a dual-homed attacker with two upstream providers, `A` and `B`. The attacker wishes to preserve a route to the victim through provider `B`. Thus, the following notation:

- `AS_v` - the victim AS
- `AS_a` - the attacker AS
- `R` - the "valid" route from `B` to `AS_v`
- `R*` - the "bogus" route from `B` to `AS_a`
- `R(X)` - the route from AS `X` to `AS_v`
- `R*(X)` - the route from AS `X` to `AS_a`
- `RR*(X)` - the set of all routes learned by AS `X` to `AS_a`

### Basic overview

The paper now presents some commonly-found topologies on the Internet, and how to achieve interception via communities. In all of these cases, `A_adv` is trying to prevent `B` from learning route `R*` (from `A`).

#### A and B peer

If A and B peer (and are not Tier-1 providers), B will most likely prefer `R*` over `R`, as it is learned from a peer rather than a paid upstream provider.

The solution, is to explicitly exclude B via the `NoExportSelect` community if available.

![](https://i.imgur.com/DGShts7.png)


#### A and B share a provider

If A and B do not have a peering relationship, there is then a (Tier-1) provider connecting them. Assuming `B` is more than 3 or 4 hops away from `A_vic`, then the AS path received by `B` from the common provider (`U1`) would be `A_adv A U1`. If `B's` upstream is a different Tier-1 (`U2`), then the route is `A_adv A U1 U2`.

The solution is to tell `U1` via communities to lower the preference of `R*`. This causes propagation of `R*` to stop at `U1`.

![](https://i.imgur.com/EBH5b4x.png)

#### A and B are both Tier-1

In this case, (since all Tier-1s are assumed to be interconnected), it is "random" whether or not `B` prefers `R` or `R*`.

The ideal solution is to use `NoExportSelect` to prevent the route from being exported to `B`. Alternatively, `NoExportAll` can be used if necessary

![](https://i.imgur.com/V00LpDS.png)


### SICO generalized

The generalized SICO attack can be described as a sequence of steps:


1. **Sample Announcement**: Announce a non-hijacked prefix, and use looking-glass to deterine internet topology
2. **Collect**: Choose a desired route to the victim. For each AS `X` in the route, determine if it could prefer `R*`. If so, it must be suppressed:
3. **Add communities**: Add targeted communities, based on the rules outlined below. Go back to step 2.
4. **Attack**: Announce the victim's prefix.

(Rather than reproduce all this here, see Table 3 in the paper)

### Specific SICO attack

Walking through the specifics of a SICO attack is illuminating. In this case, "Coloclue" is AS A, and "BIT" is AS B. The goal, then, is for BIT not to learn the malicious route.

The route from the adversary to the victim is [Adversary, BIT, KPN, Cogent, NU, Victim]

0. Export the route only via Coloclue. BIT exports the route [BIT, Coloclue, Adv].
1.  BIT and Coloclue are peers. Add communities "Coloclue: No Export to BIT". 
   Also, because the two providers peer at AMS-IX and NL-IX, add the communities "AMS-IX: No Export to BIT" and "Coloclue: No export to NL-IX" 
2. Observe the route [BIT, KPN, Coloclue, Adv]. KPN is now the problem.
3. Add community "Coloclue: no export to KPN"
4. Observe the route [BIT, NTT, Atom86, Coloclue, Adversary]. This is now the same length as the route to the victim.
5. Get very clever: Add the community "NTT: Prepend once in Europe"
6. Observe the route [BIT, NTT, NTT, Atom86, Coloclue, Adversary]. Game over.


## Countermeasures

Some commonly used simplistic countermeasures against BGP hijacks and their applicability to SICO

### Prefix filtering

Theoretically perfect, but with high administrative overhead. Somehow, nobody does this. The disappointing part is that prefix filtering for stub networks is not particularly difficult (especially with RPKI).

### Route-origin validation

Easy to work around - just prepend the victim's AS. Bye bye, RPKI.

### AS path filtering

This is a term for a broad set of techniques where routes are dropped if they have a suspect AS in the path.

One of the most common forms is to drop customer routes that have a tier-1 AS, as those must clearly be spurious.

Created partially in response to the Pilosov attack, AS path filtering does not affect SICO, as SICO does not announce a synthetic AS path.

## Potentialy effective countermeasures

### Restricting community propagation

In theory, very few ASs rely on communities propagating beyond their immediate peers. Eliminating community propagation would decrease traffic engineering features. According to the paper, at least 4000 ASs currently announce communities that propagate at least 2 ASs.

### Anomaly detection

Anomaly detection is the current state-of-the-art for detecting BGP malfeasance. The authors propose one way to extend that for watching community malfeasance.

A reasonable proposal would be to limit access to communities for untrusted AS, then, as reputation grows, grant access to more powerful communities.

## Other possible techniques

There are other ways to craft a route that will not reach from A to B:

* Invalid ROV. If the common AS between A and B enforces RPKI, but A does not, then one can announce an _intentionally invalid_ route
* AS-path filtering. If the common AS enforces AS-path filtering, then announce a route that intentionally is filtered.
* RFD: If the common AS enables damping, then intentionally flap routes to trigger damping.