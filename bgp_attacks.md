## BGP Attacks — How the Internet's Routing Gets Abused

Recall *how* BGP spreads reachability: an AS announces the prefixes it can reach, every AS
along the way prepends its own ASN to the **AS_PATH**, and each router picks a **best path**
and re-advertises it. That entire system runs on a single, unstated assumption:

> **BGP believes what it is told.** When a router hears "I originate `203.0.113.0/24`"
> or "the path to that prefix runs through these ASes," it has *no built-in way to check
> whether either claim is true.* There is no signature, no ownership record, no proof.
> A route is accepted because a neighbor said so.

That default trust is the root of every attack in this module. BGP was designed in an era
when the handful of networks on the internet knew and trusted each other. Today it
connects tens of thousands of independently operated, mutually distrustful ASes — and the
protocol still takes each of them at their word. The attacks below are all variations on a
single theme: **an AS asserts something false about routing, and the rest of the internet
believes it.**

It's useful to split the false assertions into two questions BGP never verifies:

1. **Origin:** *Is this AS actually allowed to announce this prefix?* (Attacks: prefix
   hijacking, sub-prefix hijacking.)
2. **Path:** *Is the AS_PATH a truthful record of the ASes the route really traversed?*
   (Attacks: path manipulation / forgery.)

---

### 1. Prefix Hijacking (origin hijack)

A **prefix hijack** happens when an AS announces a prefix it does not own. Because BGP has
no notion of "who is authorized to originate this address block," the false origin
announcement propagates just like a legitimate one, and neighbors accept it on faith.

Suppose **AS64500** legitimately owns and announces `203.0.113.0/24`. Now a different
network, **AS66000**, announces the *same* prefix:

```
  Legitimate origin                          Bogus origin
  ┌──────────┐                               ┌──────────┐
  │ AS64500  │  announces 203.0.113.0/24     │ AS66000  │  ALSO announces 203.0.113.0/24
  │ (owner)  │                               │ (hijack) │
  └────┬─────┘                               └────┬─────┘
       │                                          │
       └──────────► the rest of the internet ◄────┘
                    hears TWO origins for the
                    same prefix and cannot tell
                    which one is the real owner
```

Now every other AS hears two routes to `203.0.113.0/24` with two different origins. BGP has
no way to know AS64500 is the rightful owner, so each network simply applies its normal
best-path logic — shortest AS_PATH, local policy, and so on. **Every AS that ends up
preferring the route toward AS66000 will send that prefix's traffic to the attacker instead
of the legitimate owner.** Networks "closer" (in BGP terms) to the hijacker are pulled in;
the victim may not even notice, because their own local route still looks fine.

**The classic real-world example: Pakistan Telecom vs. YouTube (2008).** In an attempt to
block YouTube *domestically*, Pakistan Telecom (AS17557) announced a YouTube prefix into
BGP. The announcement leaked to its upstream provider and from there to the global internet.
Because the announcement looked like any other route, networks worldwide accepted it, and
**YouTube became unreachable for much of the planet for roughly two hours** — traffic
meant for YouTube was drawn toward Pakistan Telecom and dropped.

---

### 2. Sub-prefix Hijacking (more-specific hijack)

A prefix hijack that announces the *exact* same prefix has to win a best-path competition —
some networks will still prefer the real origin. A **sub-prefix hijack** sidesteps the
competition entirely by exploiting a hard rule of IP forwarding: **routers always forward
to the most specific (longest) matching prefix, regardless of AS_PATH length or policy.**

If the legitimate owner announces `203.0.113.0/24`, an attacker announces a **more-specific**
piece of it, such as `203.0.113.128/25`:

```
  Legitimate:  AS64500  announces  203.0.113.0/24     ( /24  = 256 addresses )
  Hijack:      AS66000  announces  203.0.113.128/25   ( /25  = the more specific half )

  A router holding both routes forwards 203.0.113.130 to the /25 —
  "longest prefix match" wins automatically. AS_PATH length never enters into it.
```

Because longest-prefix-match beats every other consideration, the more-specific route wins
**everywhere it is heard**, not just where policy happens to favor the attacker. This makes
sub-prefix hijacks more thorough and more dangerous than same-prefix hijacks.

**Real-world example: the Amazon Route 53 / MyEtherWallet hijack (2018).** Attackers
announced more-specific routes covering Amazon's authoritative DNS (Route 53) address space.
For about two hours, DNS queries for a cryptocurrency wallet site were drawn to
attacker-controlled infrastructure, which returned forged answers and steered users to a
phishing server — draining roughly $150,000 in cryptocurrency. The routing lie enabled a
DNS lie, which enabled the theft.

---

### 3. AS_PATH Manipulation (path forgery)

The first two attacks lie about the *origin*. This one lies about the *path*. Recall that
the AS_PATH is supposed to be an honest, append-only record: each AS prepends its own ASN as
the route propagates, so reading right-to-left retraces the exact chain of networks back to
the origin. **Nothing enforces that honesty.** An AS can advertise an AS_PATH that is
partly or wholly fabricated — inserting ASes that never carried the route, deleting ASes
that did, or claiming to be directly connected to a network it has never peered with.

Two consequences make this especially dangerous:

- **It defeats naive hijack detection.** A same-prefix hijacker who simply originates a
  victim's prefix is exposed as the origin (the rightmost ASN is *them*). A smarter attacker
  forges the AS_PATH so it *ends in the real owner's ASN*, making the route look as though it
  legitimately traces back to the true origin — while traffic is still diverted through the
  attacker along the way. The route looks authentic because the one field that could betray
  it has been rewritten.

- **It enables interception (man-in-the-middle), not just blackholing.** A crude hijack
  usually *drops* the stolen traffic (a denial of service). A carefully forged path lets the
  attacker **stay on a working route to the real destination**, so they can quietly
  **intercept, inspect, or modify traffic and then forward it on** — the victim's service
  still works, so nothing looks broken.

```
  What the victim's traffic is supposed to do:

     sender ───► ... ───► AS64500 (real destination)

  What a forged-path interception does:

     sender ───► AS66000 ───► ... ───► AS64500
                 (attacker      (attacker forwards on, so the
                  reads/edits    connection still "works" and
                  in transit)    the victim suspects nothing)
```

**Real-world pattern: traffic-interception incidents.** Multiple documented events — including
large-scale rerouting of traffic through networks in ways their AS_PATHs did not honestly
reflect — have quietly funneled traffic destined for banks, government sites, and major
providers through unexpected ASes before delivering it onward. Because the traffic still
arrives, these incidents can persist far longer than an outage-causing hijack before anyone
notices.

---

### Why the stakes are high

These are not edge-case curiosities. A false routing claim, accepted on trust, can cause:

- **Outages / denial of service.** Traffic for a prefix is pulled toward an AS that drops it.
  An entire service can vanish from large parts of the internet in minutes.
- **Traffic interception and eavesdropping.** Data is routed *through* an attacker who reads
  or alters it before passing it along, often without the victim noticing.
- **Credential and asset theft.** Rerouting the infrastructure *underneath* a service — DNS,
  a login endpoint, a wallet — turns a routing bug into direct financial loss (Route 53 /
  MyEtherWallet 2018).
- **Impersonation and fraud.** Hijacked address space is used to send spam, host malware, or
  impersonate trusted networks from IPs that "belong" to a reputable organization.
- **Scale and speed.** BGP propagates globally in seconds to minutes, and the damage reaches
  *every* network that believes the false route — not just the attacker's neighbors.

The common thread is that **none of these attacks require breaking into anything.** The
attacker doesn't crack a password or exploit a software bug. They simply *make a routing
announcement* — an ordinary, everyday BGP action — that happens to be false, and BGP's
trust-by-default design does the rest. The core weakness is not a flaw in any one router;
it is the absence of any mechanism to answer two basic questions:

1. **Is this AS authorized to originate this prefix?**
2. **Is this AS_PATH a truthful account of how the route actually propagated?**

Every attack above is an exploitation of the fact that, in BGP as originally designed,
**nobody is ever asked to prove either one.** The remaining modules examine how the internet
community set out to close exactly these two gaps.
