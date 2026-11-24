# Jerry

Jerry takes a fresh Debian or Ubuntu server to a working Xray-core based VPN as fast as possible.

## Why

VPN servers that are not sitting behind another proxy node have a tendency to not last long. Sooner or later the IP gets hit:

- **By a censor.** Deep packet inspection (DPI) spots the server, and the IP gets blocked.
- **By a streaming service.** The IP tends to work at first, then gets flagged after a while.
- **By Google.** Don't ask me how, they somehow can tell you are using a VPN after some time (the joy of being under OFAC sanctions).

Jerry's answer is agility. Rather than going full Solid Snake and trying to out-engineer a team full of people that get paid to block you, just spin up a new server, run Jerry, and carry on in minutes.

The hypothesis is that a censor needs far longer to catch a reasonably camouflaged server than Jerry needs to make a new one.

Jerry also does the proper (best attempt, of course, don't assume it's perfect) server setup for you, every time, so a fresh box is no longer a headache.

The primary purpose of Jerry is to outpace DPI, but the same trick works for streaming as well. Instead of paying a premium for 180-location airport services, rent a cheap server by the day in the country you need, and drop it once the show is watched.

Version 1.0 assumes a fresh server. If demand is there, I might develop it further, but note that this is a personal project I cooked with my own use cases in mind.

## What it does

Jerry runs on your own machine and does its work on the server over SSH.

It has two parts:

**A. Server setup.** Takes a fresh box and does simple (starter kit) sysadmin and security hygiene for you.
<!-- TODO: list what it actually does, once that side is built -->

**B. The VPN.** Puts an Xray config on the box, starts it, and hands you back the link (URI / QR) to connect. You can let Jerry cook the config, or give it your own; however, note that if you give it your own Xray server config, you need to build your own client config / URI.

## How it works

| Give me                        | Jerry needs                              | Jerry hands back                              |
| ------------------------------ | ---------------------------------------- | --------------------------------------------- |
| Just the setup                 | A fresh server                           | A set-up server                               |
| Setup + your own Xray config   | A path to your config                    | A running server. The link is yours to build. |
| Setup + Jerry's starter config | A few answers (in the CLI, or in a file) | A running server, plus the link / QR          |

*Why no link for your own config?*

A config can get insanely complex, and code that reads any config and parses it into a correct URI, I don't have the skill nor the time to cook.

If you go with my opinionated starter config route, Jerry asks you for some input, and generates the rest of what it needs to cook both the server and client configs. Since the exact schema of the client's config is known, Jerry can cook the URI / QR code for you.

The starter config is my own take on Xray for a 2 node set-up. If you would rather have 1 server that is harder to burn, look into a CDN fronted 3 node design; CDN fronting solves IP issues, reducing what you have to worry about to domain and traffic patterns alone.

## Jerry's Starter Config

### Note on sources

- This section reflects my understanding of Xray. It was written from the English manual + LLM translations of the Chinese manual.
- The manual appears to be written in Chinese first. If something here disagrees with it, trust the [Chinese manual](https://xtls.github.io/config/).
- 如果你看不懂中文 give the [full manual as a single text file](https://xtls.github.io/llms-full.txt) to an LLM, as described under "Use AI to configure Xray more reliably" on the [English configuration page](https://xtls.github.io/en/config/).

> Checked against Xray `<version>` on `<date>`.

### Basic configuration modules (AKA top-level keys)

Manual: <https://xtls.github.io/en/config/>

An Xray config is a single JSON object with 15 top-level keys, which the manual calls basic configuration modules. The server and the client use the same format with different content.

With the starter config, Jerry writes the server config and produces the matching client link (URI / QR code). The client application builds its own config from that link.

Jerry's starter config uses six of the 15 modules. They are numbered below in the order this section covers them.

```jsonc
{
  // WARNING: The following is NOT valid JSON/JSONC syntax. It is a documentation-only representation intended to number and categorize Xray configuration fields for Jerry's internal reference. Do not use it as an actual configuration file.

// used by Jerry
1.  "inbounds": [],
2.  "outbounds": [],
3.  "dns": {},
4.  "routing": {},
5.  "geodata": {},
6.  "log": {},

// left out
7.  "env": {},
8.  "api": {},
9.  "policy": {},
10. "stats": {},
11. "fakedns": {},
12. "metrics": {},
13. "observatory": {},
14. "burstObservatory": {},
15. "version": {}
}
```

#### Modules used by Jerry

```text
┌────────┐    inbound      ┌────────┐    outbound    ┌────────────────┐
│ client │ < ────────── >  │ SERVER │ < ────────── > │  Open Internet │
└────────┘                 └────────┘                └────────────────┘
```

##### 1. `inbounds`

Array of [InboundObject](https://xtls.github.io/en/config/inbound.html)

An inbound is the server's end of the link with the client. It defines where the server listens, which clients it accepts, and how the link is carried and disguised.

`inbounds` is an array, so a server can have several. Jerry's starter config has one: [VLESS](https://xtls.github.io/en/config/inbounds/vless.html) over [REALITY](https://xtls.github.io/en/config/transports/reality.html).

| Field                  | What it configures                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `listen`, `port`       | The address and port the server listens on.                                                                         |
| `protocol`, `settings` | The protocol this inbound speaks, and that protocol's options, such as the accepted clients.                        |
| `streamSettings`       | How the bytes travel between client and server, including the REALITY disguise.                                     |
| `tag`                  | A name for this inbound, so that other modules can refer to it.                                                     |
| `sniffing`             | Whether the server reads the destination domain from the traffic, and whether that domain replaces the destination. |

**Why Jerry uses it:** A server with no inbound has nothing for a client to connect to.

##### 2. `outbounds`

Array of [OutboundObject](https://xtls.github.io/en/config/outbound.html)

An outbound is the server's end of the link with the open internet. It defines how the server sends the client's traffic on to its destination.

`outbounds` is an array, and its order matters: the first outbound is the default, used whenever no routing rule selects another. Jerry's starter config has two:

- [Freedom](https://xtls.github.io/en/config/outbounds/freedom.html), first in the list. It connects directly to the destination and relays data in both directions.
- [Blackhole](https://xtls.github.io/en/config/outbounds/blackhole.html). It drops whatever is sent to it. The routing module uses it to refuse traffic. You can think of it as sending select connections to Sagittarius A\*: it's gone and done (the connection closes right away).

| Field                  | What it configures                                                                                  |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| `sendThrough`          | Which of the server's own IP addresses outgoing traffic leaves from.                                |
| `protocol`, `settings` | The kind of outbound, and that protocol's options.                                                  |
| `tag`                  | A name for this outbound, so that other modules can refer to it.                                    |
| `streamSettings`       | The transport configuration for this outbound. For Freedom, this is where domain resolution is set. |

Fields Jerry does not set: `mux` (client side only) and `targetStrategy` (does not apply to Freedom).

**Why Jerry uses it:** A server with no outbound has nowhere to send the client's traffic.

1. Freedom: the edge that handles server -> internet.
2. Blackhole: the server refuses to open connections to Iranian IPs.

##### 3. `dns`

> TODO: check. This entry is a stub, written before the `dns` manual page was read in full.

[DnsObject](https://xtls.github.io/en/config/dns.html)

The `dns` module is Xray's built-in DNS: it is Xray's way of resolving a domain name to an IP address.

**Note.** From what I could tell, Xray's dns runs like a function call. Another module has to ask it to resolve a name for it to kick in. In Jerry's starter config (where the client's own DNS answer is discarded and the server resolves the name itself), two modules make that call:

1. **`routing`** -> Routing makes the call for every connection whose destination is a name, checks the answer against the Iranian IPs in the geo file, and routes to Blackhole if a match is found, else to Freedom.
2. **`outbounds`** -> Routing passes the name to Freedom, then Freedom calls dns to grab the IP and send it out.

The path of a connection whose destination is a name:

```text
inbound    sniffing reads the name from the client's first bytes and sets it as the destination
routing    asks dns for the IP and compares it with the list of Iranian addresses
             Iranian      -> Blackhole, the connection is closed
             not Iranian  -> Freedom, the default outbound
Freedom    asks dns for the same name and connects to the IP it receives
```

Routing only selects the outbound and never changes the destination, so Freedom receives the name and resolves it again. The dns module caches answers by default, so the second call is expected to be answered from the cache.

A destination given as an IP address needs no lookup at either step.

Without this module, Xray uses the server's system DNS.

| Field     | What it configures                                                        |
| --------- | ------------------------------------------------------------------------- |
| `servers` | The DNS servers the module queries. Jerry sets one, supplied by the user. |

Fields left at their defaults: `hosts`, `script`, `clientIp`, `queryStrategy`, `disableCache`, `serveStale`, `serveExpiredTTL`, `disableFallback`, `disableFallbackIfMatch`, `enableParallelQuery`, `useSystemHosts`, `tag`.

**Why Jerry uses it:** So that domain names are resolved on the server, by a resolver the user chose, and so that the routing rule that refuses Iranian IPs can check a destination given as a name. I personally prefer to force DoH at the egress rather than configuring DHCP and DNS across all devices, so I went with this design.

##### 4. `routing`

> TODO: check. This entry is a stub. The `routing` manual page was covered once and is due a second pass.

[RoutingObject](https://xtls.github.io/en/config/routing.html)

The `routing` module decides which outbound each connection leaves through. For every connection, Xray checks the rules from top to bottom, and the first rule that matches decides the outbound. If no rule matches, the connection goes to the first outbound in the `outbounds` list. Routing only selects the outbound: it never changes the connection's destination.

| Field            | What it configures                                                                                 |
| ---------------- | -------------------------------------------------------------------------------------------------- |
| `domainStrategy` | Whether routing resolves a destination given as a name, so that rules matching on IP can check it. |
| `rules`          | The list of rules. Jerry has one: destinations with an Iranian IP go to the Blackhole outbound.    |

Fields Jerry does not use: `script` and `balancers`.

**Why Jerry uses it:** To refuse connections from the server to Iranian IPs.

> TODO: state the reason for this rule.
>
> TODO: decide whether a second rule should send private and local addresses (`geoip:private`) to the Blackhole.

#### Modules left out
