## BGP Refresher

**BGP (Border Gateway Protocol)** is the routing protocol that glues the internet together. It decides how traffic moves *between* independently operated networks. 

### Autonomous Systems
The internet is a collection of **Autonomous Systems (ASes)** — networks under a single administrative control (an ISP, a cloud provider, a university), each identified by a globally unique **AS number (ASN)**. For example, Google operates an Autonomous System numbered AS15169 which contains Google's global network, content delivery infrastructure, and public IP blocks. BGP's job is to exchange reachability information between ASes so each one learns how to reach every block of IP addresses (**prefixes**, e.g. `192.0.2.0/24`).

### How it works
1. **Peering / sessions.** Two routers in neighboring ASes establish a BGP session over **TCP port 179**. Once up, they exchange their full routing tables, then only incremental updates thereafter.
2. **Advertisements.** An AS *announces* the prefixes it can reach. Each announcement carries **path attributes**, most importantly the **AS_PATH** — the ordered list of ASes the route traverses.
3. **Propagation.** Each AS that accepts a route **prepends its own ASN** to the AS_PATH and re-advertises it to its neighbors. This is how reachability spreads hop-by-hop across the internet.
4. **Path selection.** A router usually hears multiple routes to the same prefix. It picks one **best path** using a decision process (local preference, then shortest AS_PATH, then other tie-breakers) and installs it, advertising only that best path onward.

BGP is a **path-vector** protocol: routes carry the full AS-level path, which lets routers detect and drop loops (an AS rejects any route already containing its own ASN).

### Example: a route propagating along a short path
Suppose **AS64500** owns the prefix `203.0.113.0/24` and wants the rest of the internet to reach it. It announces the prefix to its neighbor AS64501, which announces onward to AS64502, and so on. Each AS **prepends its own ASN** to the AS_PATH before passing the route along:

```
                announce                 announce                 announce
  203.0.113.0/24  ────────►               ────────►               ────────►
  ┌──────────┐            ┌──────────┐            ┌──────────┐            ┌──────────┐
  │ AS64500  │            │ AS64501  │            │ AS64502  │            │ AS64503  │
  │ (origin) │            │          │            │          │            │          │
  └──────────┘            └──────────┘            └──────────┘            └──────────┘

  AS_PATH as seen/stored by each AS's routing table:

  AS64500:  203.0.113.0/24  AS_PATH = [64500]                  (origin — the prefix is local)
  AS64501:  203.0.113.0/24  AS_PATH = [64500]                  (learned from origin)
  AS64502:  203.0.113.0/24  AS_PATH = [64501 64500]            (64501 prepended itself)
  AS64503:  203.0.113.0/24  AS_PATH = [64502 64501 64500]      (64502 prepended itself)
```

Reading an AS_PATH **right-to-left** traces the route back to its origin: from AS64503's view, traffic to `203.0.113.0/24` goes out to AS64502 → AS64501 → AS64500. The **leftmost** ASN is the neighbor the route was learned from; the **rightmost** is always the **origin AS**.

If AS64503 later heard the same prefix via a *different*, longer path (say through AS64502 **and** some other chain), it would compare them and, all else equal, prefer the **shorter AS_PATH**. And if a route ever arrived already containing `64503` in its path, AS64503 would **reject it as a loop**.

### eBGP vs iBGP
- **eBGP (external):** sessions between routers in *different* ASes — the actual inter-domain exchange.
- **iBGP (internal):** sessions between routers *within the same* AS, used to carry externally-learned routes across the network.

### Policy and business relationships
BGP path selection is driven less by shortest-path than by **policy** reflecting commercial relationships:
- **Customer** (pays you) → **Peer** (settlement-free) → **Provider** (you pay them).
- Networks prefer routing through customers over peers over providers, and generally don't re-advertise routes in ways that would make them transit traffic for free (**valley-free routing**).

### More BGP resources
https://datatracker.ietf.org/doc/html/rfc4271 - RFC covering BGP
https://www.kentik.com/kentipedia/bgp-routing/ - BGP tutorial