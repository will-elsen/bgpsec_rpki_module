### BGP Refresher

**BGP (Border Gateway Protocol)** is the routing protocol that glues the internet together. It decides how traffic moves *between* independently operated networks. It's an **inter-domain** protocol — routing *within* a single network is handled by IGPs like OSPF or IS-IS.

#### Autonomous Systems
The internet is a collection of **Autonomous Systems (ASes)** — networks under a single administrative control (an ISP, a cloud provider, a university), each identified by a globally unique **AS number (ASN)**. BGP's job is to exchange reachability information between ASes so each one learns how to reach every block of IP addresses (**prefixes**, e.g. `192.0.2.0/24`).

#### How it works
1. **Peering / sessions.** Two routers in neighboring ASes establish a BGP session over **TCP port 179**. Once up, they exchange their full routing tables, then only incremental updates thereafter.
2. **Advertisements.** An AS *announces* the prefixes it can reach. Each announcement carries **path attributes**, most importantly the **AS_PATH** — the ordered list of ASes the route traverses.
3. **Propagation.** Each AS that accepts a route **prepends its own ASN** to the AS_PATH and re-advertises it to its neighbors. This is how reachability spreads hop-by-hop across the internet.
4. **Path selection.** A router usually hears multiple routes to the same prefix. It picks one **best path** using a decision process (local preference, then shortest AS_PATH, then other tie-breakers) and installs it, advertising only that best path onward.

BGP is a **path-vector** protocol: routes carry the full AS-level path, which lets routers detect and drop loops (an AS rejects any route already containing its own ASN).

#### eBGP vs iBGP
- **eBGP (external):** sessions between routers in *different* ASes — the actual inter-domain exchange.
- **iBGP (internal):** sessions between routers *within the same* AS, used to carry externally-learned routes across the network.

#### Policy and business relationships
BGP path selection is driven less by shortest-path than by **policy** reflecting commercial relationships:
- **Customer** (pays you) → **Peer** (settlement-free) → **Provider** (you pay them).
- Networks prefer routing through customers over peers over providers, and generally don't re-advertise routes in ways that would make them transit traffic for free (**valley-free routing**).

#### Why security matters (RPKI / BGPsec context)
BGP was designed with **implicit trust** — it does not natively verify that an AS is actually authorized to announce a prefix, or that an AS_PATH is genuine. This enables:
- **Prefix hijacking:** an AS announces a prefix it doesn't own, diverting traffic.
- **Route leaks:** routes are re-advertised against policy, redirecting traffic through unintended paths.
- **Path manipulation:** a forged AS_PATH.

Security extensions address these gaps:
- **RPKI (Resource Public Key Infrastructure):** cryptographically binds prefixes to the ASNs authorized to originate them via signed **ROAs (Route Origin Authorizations)**, enabling **Route Origin Validation (ROV)** — catching hijacks where the *origin* AS is wrong.
- **BGPsec:** cryptographically signs the AS_PATH itself, so each hop can verify the full path was not forged — addressing what ROV alone cannot.
