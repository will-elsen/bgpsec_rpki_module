## BGP Trace Exercise (BGP Refresher Module)

**Goal:** trace the path from your own public IP to a well-known destination (Google,
AS15169) and reconstruct the AS-level route — first as *packets actually flow*
(traceroute), then as *BGP really sees it* (looking glasses).

> **Key idea up front:** you don't run BGP — your ISP does. So you can't read the true
> BGP AS-path directly from your machine. You approximate it two ways: (1) traceroute +
> AS mapping shows the **forwarding path**, and (2) public looking glasses / route
> collectors show the **real BGP AS_PATH** from routers that actually speak BGP.
> The two can differ (BGP routes by policy, not hop count; reverse paths are often
> asymmetric). Combining them gets you very close to the truth.

---

### Step 1 — Find your ASN (where your path begins)

In a terminal window, run:

```bash
curl -s https://ipinfo.io/json
```

This returns your public IP and the network that owns it. If you see HTML instead of
JSON, use the explicit `/json` endpoint above (or add `-H "Accept: application/json"`).

On **Windows PowerShell**, the built-in `curl` is an alias for `Invoke-WebRequest`, so
call real curl explicitly:

```powershell
curl.exe -s https://ipinfo.io/json | ConvertFrom-Json
```

Look at the `"org"` field — it looks like `AS7922 Comcast Cable Communications`. That
**ASN is where your route begins** in BGP terms (your ISP's AS, not you personally).

Record it:

```
My public IP:   ____________________
My ISP ASN:     AS__________  (name: ______________________)
```

Grab just the org line if you like:

```bash
curl -s https://ipinfo.io/org      # -> "AS#### <ISP name>"
```

---

### Step 2 — Identify the destination ASN

Google's public DNS (`8.8.8.8`) is a convenient, well-connected target.

- **Destination IP:** `8.8.8.8`
- **Destination ASN:** `AS15169` (Google LLC)

You can confirm any IP's ASN at [bgp.he.net](https://bgp.he.net) (search the IP) or:

```bash
curl -s https://ipinfo.io/8.8.8.8/json
```

---

### Step 3 — Traceroute the forwarding path (with ASNs)

**`mtr` with the `-z` flag** annotates each hop with its ASN — the best interactive view:

```bash
mtr -zbw 8.8.8.8
#   -z  show AS number per hop
#   -b  show both hostname and IP
#   -w  report mode (runs a fixed number of cycles, then prints a table)
```

Plain traceroute alternatives:

```bash
traceroute 8.8.8.8        # Linux / macOS
tracert 8.8.8.8           # Windows (no ASN column)
```

On **Windows**, `tracert` gives hops but no ASN. Options:
- Run `mtr -z` inside **WSL**, or
- Paste your traceroute output into a web AS-path mapper, or
- Skip to Step 4 and read the BGP path directly from a looking glass.

**What to expect / caveats:**
- Google (AS15169) peers extremely widely, so you'll often reach it in just **2–4 ASes**.
- `* * *` gaps are normal — many core routers don't respond to TTL-expired probes.
- The forwarding path can differ from the BGP AS_PATH, and the return path may differ again.

Record the AS hops you observe:

```
Hop ASNs (forwarding path):
  AS________  (my ISP)
  AS________  (___________________)
  AS________  (___________________)
  AS15169     (Google)
```

---

### Step 4 — Read the *real* BGP AS_PATH (looking glasses)

Now get the genuine BGP view from routers that actually run BGP:

1. **[bgp.he.net](https://bgp.he.net)** — search for `8.8.8.0/24` or `AS15169`. You'll see
   Google's announced prefixes, its peers, and AS_PATHs observed across the internet.
2. **[RIPE Stat](https://stat.ripe.net)** — enter the prefix/ASN for BGP routing data,
   AS_PATH visualizations, and history.
3. **Public looking glasses** (e.g. [lg.he.net](https://lg.he.net)) — run
   `show ip bgp 8.8.8.0/24` on a real BGP router and read the AS_PATH field directly.
4. **Route collectors** — [RIPE RIS](https://ris.ripe.net) and
   [RouteViews](http://www.routeviews.org) expose global BGP tables if you want raw
   data or to script against it.

Read an AS_PATH **right-to-left**: the **rightmost** ASN is the origin (Google, 15169),
the **leftmost** is the AS closest to the collector. Example AS_PATH you might see:

```
AS_PATH:  6939 15169
          ^      ^
          |      └─ origin AS (Google)
          └──────── Hurricane Electric (transit/peer)
```

---

### Step 5 — Reconstruct the AS-level route

Line up what you found:

```
Start:        AS________  (my ISP, from Step 1)
              ↓  via transit / peering
Middle:       AS________ , AS________  (from traceroute + looking glass)
              ↓
Destination:  AS15169  (Google, the origin AS)
```

Questions to reflect on:
- Does the **forwarding path** (Step 3) match the **BGP AS_PATH** (Step 4)? Where do
  they diverge, and why might that be (policy vs. hop count, asymmetric return path)?
- How many ASes did it actually take? (Often surprisingly few for Google.)
- Which AS is the **origin**? That's the ASN RPKI/ROV would check for authorization to
  announce the prefix — the link back to this repo's RPKI/BGPsec modules.

---

### Appendix — Quick command reference

```bash
# Who am I (BGP-wise)?
curl -s https://ipinfo.io/json          # full JSON for your IP
curl -s https://ipinfo.io/org           # just "AS#### <ISP>"

# Forwarding path to Google DNS, with ASNs
mtr -zbw 8.8.8.8                         # Linux/macOS/WSL
tracert 8.8.8.8                          # Windows (no ASN column)

# Look up any IP's ASN
curl -s https://ipinfo.io/8.8.8.8/json

# Real BGP AS_PATH: bgp.he.net / stat.ripe.net / lg.he.net (web)
```

**Reserved-doc note:** `8.8.8.8`/`AS15169` are real and safe to query. If you write up
hypothetical examples, prefer documentation ASNs (64496–64511, 65536–65551) and prefixes
(`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`) so nothing collides with real
assignments — same convention used in `bgp_refresher.md`.
