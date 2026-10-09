# Fortigate-Routing-Table-Analysis

# FortiGate Routing Table Analysis — `get router info routing-table database`

## Command shown

```bash
get router info routing-table database
```

This command displays routes in FortiGate's routing database for `VRF=0` (the default routing instance).

---

## Route codes shown

```text
S = Static route
C = Connected route
*> = Selected/active best route
```

- `S` means an administrator manually configured the route.
- `C` means the route exists because an IP subnet is configured directly on a FortiGate interface.
- `*>` means the route is installed and selected for forwarding traffic.

---

## Routing-table entries

```text
S    *> 0.0.0.0/0 [10/0] via 100.66.0.254, port2, [1/0]
     *> 0.0.0.0/0 [10/0] via 100.66.0.254, port3, [2/0]

C    *> 10.0.11.0/24 is directly connected, port4
C    *> 10.0.12.0/24 is directly connected, port5
C    *> 10.0.13.0/24 is directly connected, port6
C    *> 100.65.0.0/24 is directly connected, port2
C    *> 100.66.0.0/24 is directly connected, port3
C    *> 192.168.0.0/16 is directly connected, port1
```

---

# Default Route

## How to identify the default route

A default route uses:

```text
0.0.0.0/0
```

The `/0` prefix means it matches **every IPv4 destination**.

Your table has two default routes:

```text
0.0.0.0/0 via 100.66.0.254, port2
0.0.0.0/0 via 100.66.0.254, port3
```

Both routes use the same next-hop IP address, `100.66.0.254`, but exit through different interfaces:

| Default route | Next hop | Egress interface |
|---|---|---|
| Default route 1 | `100.66.0.254` | `port2` |
| Default route 2 | `100.66.0.254` | `port3` |

Note: The correct networking term is **default route**, not default port. The egress interfaces for the default routes are `port2` and `port3`.

A default route is only selected when FortiGate has no more-specific matching route for the destination. For example, traffic to `8.8.8.8` will use one of these default routes, while traffic to `10.0.12.25` uses the directly connected `10.0.12.0/24` route through `port5`.

---

# Best Route Selection

## Can this table identify the best route?

Yes.

The `*>` marker indicates a selected/best active route. In this output, every listed route has `*>`, meaning it is installed in the routing table and can be used for forwarding.

FortiGate route selection follows this main order:

```text
1. Longest prefix match
2. Lowest administrative distance
3. Lowest metric/cost
4. Lowest route priority
5. ECMP if equal routes remain
```

Example using this table:

```text
Destination: 10.0.12.50
Matching routes:
- 10.0.12.0/24 through port5
- 0.0.0.0/0 through port2 or port3

Selected route:
- 10.0.12.0/24 through port5
```

Reason:

```text
10.0.12.0/24 is more specific than 0.0.0.0/0.
```

Therefore, FortiGate sends traffic for `10.0.12.0/24` out of `port5`, not through either default route.

For traffic to an internet address such as `8.8.8.8`, no more-specific route appears in this table. FortiGate can use the two `0.0.0.0/0` default routes.

Useful verification command:

```bash
diagnose ip route match 8.8.8.8
```

---

# Administrative Distance, Metric, and Priority

## What does `[10/0]` mean?

```text
[10/0]
```

This means:

| Value | Meaning |
|---:|---|
| `10` | Administrative distance |
| `0` | Metric |

For both default routes:

```text
[10/0]
```

- Administrative distance is `10`
- Metric is `0`

A static route normally has an administrative distance of `10`.

Lower administrative distance is preferred when FortiGate learns the same destination through different route sources.

Example:

```text
Static route AD 10
OSPF route AD 110
```

If both routes target the same prefix length and destination, FortiGate prefers the static route because `10` is lower than `110`.

## Where is the priority?

The final bracket contains route priority and weight:

```text
[1/0]
[2/0]
```

Format:

```text
[priority/weight]
```

Your routes are:

| Route | Priority | Weight |
|---|---:|---:|
| `0.0.0.0/0` via `port2` | `1` | `0` |
| `0.0.0.0/0` via `port3` | `2` | `0` |

Important rule:

```text
Lower priority value = higher preference.
```

Therefore:

```text
port2 priority 1 = preferred default route
port3 priority 2 = backup/secondary default route
```
