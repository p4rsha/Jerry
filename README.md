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

Jerry's starter config uses five of the 15 modules. They are numbered below in the order this section covers them.

```jsonc
{
  // WARNING: The following is NOT valid JSON/JSONC syntax. It is a documentation-only representation intended to number and categorize Xray configuration fields for Jerry's internal reference. Do not use it as an actual configuration file.

// used by Jerry
1.  "inbounds": [],
2.  "outbounds": [],
3.  "dns": {},
4.  "routing": {},
5.  "geodata": {},

// Left out to default
6.  "log": {},
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
┌──────┐  client inbound  ┌────────┐  client outbound  ┌────────┐  server outbound  ┌───────────────┐
│ apps │ < ──────────── > │ CLIENT │ < ───────────── > │ SERVER │ < ───────────── > │ Open Internet │
└──────┘                  └────────┘   server inbound  └────────┘                   └───────────────┘
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

The `routing` module decides which outbound each connection leaves through. For every connection, Xray checks the rules **from top to bottom, and the first rule that matches decides the outbound.**

**If no rule matches, the connection goes to the first outbound in the `outbounds` list.**

*Note:*
> In JSON, the order of keys inside an object does not matter, but the order of items inside an array does. `outbounds` is an array, and its first item is the default outbound. In Jerry's starter config **Freedom must stay first**: if Blackhole came first, every connection that matched no rule would be dropped.

Routing only selects the outbound: it never changes the connection's destination.

| Field            | What it configures                                                                          |
| ---------------- | ------------------------------------------------------------------------------------------- |
| `domainStrategy` | What routing does when the destination is a name and a rule needs an IP.                    |
| `rules`          | The list of rules. Each rule is a set of conditions and the outbound to use when they hold. |

Fields Jerry does not use: `script` and `balancers`.

`domainStrategy` has three values:

- `AsIs` (the default): routing never calls the dns module. A rule that needs an IP cannot match a name.
- `IPIfNonMatch`: routing first goes through the whole list with the name. If nothing matched, it calls the dns module and goes through the list again with the IP.
- `IPOnDemand`: routing calls the dns module as soon as it reaches a rule that needs an IP.

Jerry's starter config has one rule:

```text
condition   the destination IP is on the Iran list (geoip:ir)
send to     the Blackhole outbound, named by its tag
```

Jerry's one rule decides on the IP, so routing needs an IP for every connection. When the destination arrives as a name, routing asks the dns module to resolve it and then runs the check. The setting for that is `domainStrategy: "IPOnDemand"`.

**Why Jerry uses it:** To refuse connections from the server to Iranian IPs.

> TODO: state the reason for this rule.
>
> TODO: decide whether a second rule should send private and local addresses (`geoip:private`) to the Blackhole.

##### 5. `geodata`

> TODO: check. This entry is a stub. The `geodata` manual page has not been read in full, and its values are not decided.

[GeodataObject](https://xtls.github.io/en/config/geodata.html)

The `geodata` module keeps Xray's geodata files up to date. On a schedule, it downloads new copies of the files and reloads them, without restarting Xray. These files hold the address lists that rules such as `geoip:ir` read. Without this module, the files stay as they were when Xray was installed.

| Field    | What it configures                                                               |
| -------- | -------------------------------------------------------------------------------- |
| `cron`   | The schedule, as a cron expression. Without it, no scheduled task runs.          |
| `assets` | The files to download and replace, each with a download address and a file name. |

Fields Jerry does not set: `outbound` (the outbound used for the downloads; when left out, they follow the routing rules).

**Why Jerry uses it:** So that the Iran address list read by the routing rule stays current while the server is running.

> TODO: choose the schedule and the download source.
>
> TODO: the manual warns against using this module on low-memory devices. Check it on the smallest server Jerry targets.
> TODO: `assets` can only replace files that already exist. Confirm the installer has placed `geoip.dat` before the first scheduled run.

#### Modules left out

*Disclaimer:*
> I did not read the manuals on these keys myself, it's a synthesis by AI. It's basically my 受け売り of what an LLM told me, so refer to it at your own risk.

##### 6. `log`

[LogObject](https://xtls.github.io/en/config/log.html)

**What it does:** Controls how Xray writes its two logs: the access log, which records connections, and the error log, which records problems. Each can be sent to a file, sent to standard output, or turned off.

**Left out, so the defaults apply:** Both logs are written to standard output, which the system journal collects when Xray runs as a service. The error log records warnings and errors. The access log is on.

**Why Jerry leaves it out:** The defaults need no setup, which keeps the starter config small. The journal limits its own size, so there are no log files to rotate. A Jerry server is supposed to be short-lived, so the connection records do not outlive it. To keep no connection records at all, add the module with `"access": "none"`.

> TODO: confirm on a real server that Xray starts with `log` left out and that its output reaches the journal.

##### 7. `env`

[EnvObject](https://xtls.github.io/en/config/env.html)

**What it does:** Sets environment variables for the Xray process from inside the config file. These variables adjust low-level behavior, such as where Xray looks for its data files and how it sizes its buffers.

**Left out, so the defaults apply:** Xray sets no variables of its own. It runs with its built-in low-level settings, plus whatever the service that starts it already provides.

**Why Jerry leaves it out:** These are low-level settings for special cases. The one that matters to Jerry, the location of the geodata files, already matches where the installer puts them.

##### 8. `api`

[ApiObject](https://xtls.github.io/en/config/api.html)

**What it does:** Opens a remote-control interface to the running Xray. Another program can use it to add or remove users, inbounds, outbounds and routing rules, and to read traffic statistics, all without restarting Xray.

**Left out, so the defaults apply:** No interface is opened. The only way to change the server is to edit the config and restart Xray.

**Why Jerry leaves it out:** It exists for tools that manage many users on a long-running server. A Jerry server has one user and a config that is written once. Leaving it out also means one less service listening on the server.

##### 9. `policy`

[PolicyObject](https://xtls.github.io/en/config/policy.html)

**What it does:** Sets rules per user level: how long a connection may take to set up, how long it may sit idle before it is closed, how much is buffered, and whether traffic is counted. Every user and every outbound carries a level number, and that number selects which set of rules applies.

**Left out, so the defaults apply:** Every connection gets Xray's default limits, for example 60 seconds to complete a handshake and 300 seconds of idle time.

**Why Jerry leaves it out:** It tunes how connections behave and has little to do with getting past DPI. Jerry's starter config is meant someone using xray for personal VPN, so has one user at level 0. If you wanna charge someone for open source code, you've come to the wrong repo.

##### 10. `stats`

[StatsObject](https://xtls.github.io/en/config/stats.html)

**What it does:** Switches on Xray's internal traffic counters, which add up the bytes sent and received per user, per inbound and per outbound. It has no settings of its own: the `policy` module chooses what is counted, and the `api` module is how the numbers are read out.

**Left out, so the defaults apply:** Nothing is counted.

**Why Jerry leaves it out:** Jerry prefers counting with his fingers.

##### 11. `fakedns`

[FakeDnsObject](https://xtls.github.io/en/config/fakedns.html)

**What it does:** Answers DNS questions with made-up IP addresses from a reserved range, and remembers which name each made-up address stands for. When a connection to one of those addresses arrives, Xray translates it back into the name. It is used on the client side, with TUN or transparent proxy setups, where the name would otherwise be lost.

**Left out, so the defaults apply:** No made-up addresses are handed out. Names are learned by sniffing instead.

**Why Jerry leaves it out:** The server learns names by sniffing, which is enough.

##### 12. `metrics`

[MetricsObject](https://xtls.github.io/en/config/metrics.html)

**What it does:** Serves Xray's internal numbers (traffic counters, and memory and performance profiles) at a web address, so that a monitoring tool can read them and draw graphs.

**Left out, so the defaults apply:** No such address is served.

**Why Jerry leaves it out:** It is for watching a long-running server on a dashboard.

##### 13. `observatory`

[ObservatoryObject](https://xtls.github.io/en/config/observatory.html)

**What it does:** Tests a set of outbound proxies at a fixed interval to see which are reachable and how quickly they respond. Other parts of Xray, such as the balancers in `routing`, use the results to pick an outbound.

**Left out, so the defaults apply:** No outbound is tested.

**Why Jerry leaves it out:** It is for configs that choose between several outbound proxies. Jerry's server has a single way out to the internet and does not use balancers.

##### 14. `burstObservatory`

[BurstObservatoryObject](https://xtls.github.io/en/config/observatory.html)

**What it does:** The same job as `observatory`, with the tests spread over random moments so that they are harder to recognize as a pattern. A config uses one of the two, not both.

**Left out, so the defaults apply:** No outbound is tested.

**Why Jerry leaves it out:** The same reason as `observatory`.

##### 15. `version`

[Configuration overview](https://xtls.github.io/en/config/)

**What it does:** States which Xray versions may run this config, as a lowest and a highest version. It exists for shared configs, so that one does not run by accident on a version it was not written for.

**Left out, so the defaults apply:** No version check is made. The config runs on whatever Xray is installed.

**Why Jerry leaves it out:** Well , basically because I ran out of time for V.1. Jerry uses PQ encryption, so, setting appropriate min version would be the smart thing to do. BUT , it's taking me forever to get this out, so shipping without it.

### `inbounds`

Manual: [Inbound](https://xtls.github.io/en/config/inbound.html)

An inbound is the end of a link where a node accepts connections. The same word means a different link on each side:

| Node   | This inbound accepts connections from | This outbound opens connections to |
| ------ | ------------------------------------- | ---------------------------------- |
| Client | apps on the device                    | the server                         |
| Server | client                                | open internet                      |

The client's outbound and the server's inbound are the two ends of one link, so their settings have to match. Jerry writes the server's end into the config file and hands over the client's end as the link (URI / QR code).

Each item in the `inbounds` array is one InboundObject with seven fields:

```jsonc
{
  // Not valid JSON: the numbers are for reference only.
1.    "inbounds": [
        {          // Server's Inbound | Client's Inbound
1.1       "listen": "<READ FROM THE SERVER: IPv4v6> | 127.0.0.1",
1.2       "port": "443 | <chosen by the client app on >",
1.3       "protocol": "vless | <chosen by the client app>",
1.4       "settings": {},
1.5       "streamSettings": {},
1.6       "tag": "identifier",
1.7       "sniffing": {
            "enabled": true,
            "destOverride": ["http", "tls"]
          }
        }
      ]
}
```

#### 1.1 `listen`

> `listen`: an IP address the node holds. On the server, read from the server itself. On the client, `127.0.0.1`.

The IP address a node binds to and accepts connections on. In the server's config, it is where the client connects. In the client's config, it is where the apps on the device connect, usually `127.0.0.1`.

*FYI:*

- **Default:** `0.0.0.0`, which means every IP address the server holds.
- **Also accepts:** a Unix socket path in place of an address. If you fancy putting Xray behind a reverse proxy on your server, this is the option to look at.
- **Note:** on a UDP inbound, a server that holds several IP addresses should bind to one specific address.
  - With TCP, the handshake settles both ends of a connection: the client's address, and the server address the client contacted. Every reply leaves from that server address.
  - UDP has no handshake, so nothing is settled. Each reply needs a source address chosen for it.
  - When the server listens on `0.0.0.0`, the kernel has to make that choice without knowing which address the client contacted, and it silently uses its default one. Nothing fails on the server, but the client ignores a reply that comes from an address it never wrote to.
  - Binding to one address leaves the kernel nothing to choose.
  - Jerry's inbound is TCP, so this does not affect the starter config.

**Jerry sets:** the server's inbound only. Its `listen` is read from the server.

The client's inbound is written by the client application, not by Jerry. It is usually `127.0.0.1`.

The public address the user inputs is used for the SSH setup and for the client's outbound link. Jerry does not use it for the server's inbound: it runs a command on the server to read the address, so that servers behind NAT do not break.

*Note:*

On the server side, Jerry pins the IP address Xray listens on. This breaks the config if the address the server itself holds ever changes, for example after a reboot. If that happens, change this field to `0.0.0.0`.

> TODO: state why Jerry binds to one address instead of `0.0.0.0`.

#### 1.2 `port`

> `port`: number | "env:variable" | string

The port a node listens on, at the address given in `listen`. In the server's config, it is the port the client connects to. In the client's config, it is the port the apps on the device connect to.

*FYI:*

- **Accepts:** one port (`443`), a range (`"5-10"`), or a list that mixes both (`"11,13,15-17"`).
- **Also accepts:** `"env:NAME"`, which reads the port from an environment variable.
- **Note:** each port in a range gets its own listening socket, so large ranges are costly. The manual says problems may start as the count nears four digits.

**Jerry sets:** `443` on the server's inbound. It is the port HTTPS uses, and the REALITY disguise presents the server as an ordinary HTTPS site.

The client's inbound port is chosen by the client application, not by Jerry. The manual's sample uses `1080`.

The same `443` appears in the client's outbound link, right after the server's public address.

> TODO (setup side): the port has to be open in the server's firewall, and on some providers in the provider's firewall too.

#### 1.3 `protocol`

> `protocol`: one protocol name

The protocol an inbound speaks: the agreed format in which the two ends of the link exchange bytes. It is chosen once per inbound, and the choice decides the shape of `settings` (1.4).

*FYI:*

The manual lists eleven inbound protocols. They fall into two groups, by who is on the other end:

| The other end is               | Protocols                                                                    |
| ------------------------------ | ---------------------------------------------------------------------------- |
| another proxy node             | `vless`, `vmess`, `trojan`, `shadowsocks`, `wireguard`, `hysteria`, `masque` |
| an app or the operating system | `socks`, `http`, `tun`, `tunnel`                                             |

**Jerry sets:** `vless` on the server's inbound. [VLESS](https://xtls.github.io/en/config/inbounds/vless.html) with REALITY and Vision is the setup the Xray manual recommends for a client that connects directly to its server. Vision and VLESS Encryption are both set in `settings`.

The client's outbound uses `vless` as well, since it is the other end of the same link. The client's inbound uses a protocol from the second group, chosen by the client application.

#### 1.4 `settings`

Manual: [VLESS inbound](https://xtls.github.io/en/config/inbounds/vless.html)

`settings` holds the options of the protocol chosen in 1.3, so its shape changes with the protocol. This section covers the `vless` shape.

VLESS is a lightweight protocol that links an Xray client to an Xray server. It identifies each user by a UUID.

With `vless`, `settings` has four fields :

1. `users`
2. `flow`
3. `decryption`
4. `fallbacks`

```jsonc

{
  // Not valid JSON: the numbers are for reference only.
1.4       "settings": {
1.4.1       "users": [
              {
                "id": "<GENERATED: xray uuid>",
                "level": 0 // It's personal use , so , Jerry cooks 1 user , all users are level 0
              }
            ],
1.4.2       "flow": "xtls-rprx-vision", // XTLS Vision. It hides the TLS inside TLS traffic pattern through padding. 
1.4.3       "decryption": "<GENERATED: xray vlessenc, the server string>"
// 1.4.4 fallbacks: not written , because it cannot be used with decryption that is not none
          }
}
```

##### 1.4.1 `users`

> `users`: one user, with an `id` generated on the server and `level` 0.

The list of users the server accepts. Each user is one identity that a client can connect with.

**Jerry sets:**

- `id`: a random UUID, generated on the server with `xray uuid`. The same UUID goes into the client's link.
- `level`: `0`. Jerry creates one user for personal use, and the `policy` module is left out, so there is only one level.

| Field     | What it is                                                                                                                |
| --------- | ------------------------------------------------------------------------------------------------------------------------- |
| `id`      | The user's ID, which works as the VLESS password. A UUID, or any string shorter than 30 bytes, which Xray maps to a UUID. |
| `level`   | The user level, which selects the `policy` rules that apply. Default `0`.                                                 |

Fields Jerry does not set: `email` (nothing reads the label), `flow` (set once in 1.4.2 instead), and `reverse` (the server is not used as a reverse proxy).

- **Note:** a custom string and the UUID it maps to are the same user. A string that is easy to guess is therefore a weak ID.

*FYI:*

Each user can carry five fields. Here are the ones Jerry does not write:

| Field     | What it is                                                                                                                |
| --------- | ------------------------------------------------------------------------------------------------------------------------- |
| `email`   | A label for this user in logs and statistics.                                                                             |
| `flow`    | The flow control mode for this user. Left empty, the user takes the value of `settings.flow` (1.4.2).                     |
| `reverse` | Turns this user's connections into a reverse proxy tunnel, and disables normal use for that user.                         |

##### 1.4.2 `flow`

> `flow`: `xtls-rprx-vision`, set once for every user.

The flow control mode for the inbound. It decides how the traffic inside the link is handled, and with it whether Vision is used. The value set here is the default for every user whose own `flow` is empty.

**Jerry sets:** `xtls-rprx-vision`.

- **What Vision does:** the client's traffic is usually TLS already, so carrying it inside REALITY produces TLS inside TLS, which has a recognizable pattern of sizes and timing. Vision pads the inner TLS handshake to hide that pattern. After the handshake, when the inner traffic is TLS 1.3, it passes the rest through as it is, without wrapping it a second time.
- **Why here and not on the user:** a non-empty value at this level cannot be switched off from a user entry, because an empty user `flow` falls back to it. Vision is therefore on for every user.
- **The client's link** carries the same value, since both ends of the link have to use the same mode.

*FYI:*

| Value              | What it means                        |
| ------------------ | ------------------------------------ |
| empty, or left out | A regular TLS proxy, with no Vision. |
| `xtls-rprx-vision` | Vision, as described above.          |

- **Note:** Vision only works in certain setups: over TCP with TLS or REALITY, or with VLESS Encryption turned on.

##### 1.4.3 `decryption`

> `decryption`: the server string printed by `xray vlessenc`, from its X25519 pair.

The setting for VLESS Encryption, an extra layer of encryption inside the link, underneath REALITY. It encrypts the VLESS header and the handshake of the traffic inside (AKA : the TLS handshake between your browser and the website is PQ encrypted while it crosses the Client <-> Server edge), using a post-quantum key exchange. This field cannot be left empty: to turn the layer off, it has to say `"none"`.

**Jerry sets:** the server string from `xray vlessenc`, generated on the server. The matching client string goes into the client's link.

- **Which pair:** `xray vlessenc` prints two pairs of strings, one built on X25519 and one on ML-KEM-768. Jerry uses the X25519 pair.

- **Why X25519 instead of ML-KEM-768:**

At first glance, it can be confusing, because why feed X25519 to `decryption` when it's supposed to do PQ encryption , which is ML-KEM territory. BUT , even with X25519, the Vless header IS PQ encrypted , BUT, it **ONLY PROTECTS AGAINST RECORDING -> LATER DECRYPTION GIVEN QC**. This pair only decides how the client checks that it is talking to the real server. The key exchange that protects the data is post-quantum with either pair. The ML-KEM-768 pair would make the client's link about 1,600 characters longer, for a benefit that only matters if the client string leaks and a quantum computer exists.

*Long Story Short:*
> Set `decryption` to a ML-KEM-768 once QC's exist. Before that , you good using X25519, no one should be able to record your traffic and retroactively go decrypt your shit.

- **Trade-offs:** the client needs a recent Xray, from August 2025 or later, and `fallbacks` cannot be used together with it.

*FYI:*

| Value                         | What it means            |
| ----------------------------- | ------------------------ |
| `"none"`                      | VLESS Encryption is off. |
| a string from `xray vlessenc` | VLESS Encryption is on.  |

**Note:** the string is a list of blocks joined by dots: the handshake method, how the traffic should look, a ticket lifetime, padding settings, and at the end the key that proves the server's identity. The manual advises generating it with `xray vlessenc` instead of writing it by hand.

##### 1.4.4 `fallbacks`

> `fallbacks`: not written.

A list of places to send a connection that turns out not to be valid VLESS. On an inbound that uses ordinary TLS, a visitor who fails the VLESS check is handed to another service, usually a real web server. Someone probing the port then sees a normal website, not a closed connection.

**Jerry sets:** nothing. The field is left out.

- **Why:** fallbacks belong to inbounds that use TCP with ordinary TLS, and the manual says other setups must not have them. Jerry's inbound uses REALITY, which does the same job its own way: a connection that fails REALITY's check is forwarded to the `target` site (see 1.5).
- **Also:** fallbacks cannot be combined with VLESS Encryption, which Jerry turns on in 1.4.3.

*FYI:*

- **Left out, so the default applies:** a connection that fails the VLESS check is closed.

#### 1.5 `streamSettings`

Manual: [Transport](https://xtls.github.io/en/config/transport.html)

`streamSettings` is the transport configuration. It decides how the bytes of the link are carried between the two nodes, underneath the VLESS protocol. Both ends of a link have to use matching settings, which is why most of these choices also appear in the client's link.

The manual groups the settings into three layers:

| Layer              | Fields                                                  | Jerry sets                          |
| ------------------ | ------------------------------------------------------- | ----------------------------------- |
| Transport method   | `method`, plus the settings object of the chosen method | `raw`: plain TCP with nothing added |
| Transport security | `security`, plus the settings object of the chosen kind | `reality`                           |
| Extra settings     | `finalmask`, `sockopt`                                  | both empty                          |

```jsonc
{
  // Not valid JSON: the numbers are for reference only.
1.5       "streamSettings": {
1.5.1       "method": "raw",
1.5.2       "rawSettings": {
              "acceptProxyProtocol": false,
              "header": { "type": "none" }
            },
1.5.3       "security": "reality",
1.5.4       "realitySettings": { "<12 fields, see 1.5.4>" },
1.5.5       "finalmask": {},
1.5.6       "sockopt": {}
          }
}
```

Fields Jerry does not write: the settings objects of the methods and the security kind it does not use (`xhttpSettings`, `kcpSettings`, `grpcSettings`, `wsSettings`, `httpupgradeSettings`, `hysteriaSettings`, `masqueSettings`, `tlsSettings`). Xray only reads the object that matches the chosen value.

*FYI:*
If you wanna cook a CDN fronted config, look into xhttp.

##### 1.5.1 `method`

> `method`: `raw`.

The transport method: what carries the bytes of the link between the two nodes. Exactly one is chosen, and both ends have to choose the same one.

**Jerry sets:** `raw`, which is plain TCP with nothing added. It is also the default.

- **Why:** the client connects straight to the server, with nothing in between. The main use case beyond raw, for multi node designs. REALITY and Vision both work over `raw`.
- **The client's link** names the same method.

*FYI:*

The manual lists eight methods. They fall into four groups:

| Group                            | Methods                                     | When it is used                                                              |
| -------------------------------- | ------------------------------------------- | ---------------------------------------------------------------------------- |
| TCP, nothing added               | `raw`                                       | The two nodes talk to each other directly.                                   |
| TCP, dressed as HTTP             | `xhttp`, `websocket`, `httpupgrade`, `grpc` | Something in the middle, such as a CDN or a reverse proxy, only passes HTTP. |
| UDP, with its own delivery logic | `mkcp`, `hysteria`                          | Speed on a poor line, at the cost of standing out.                           |
| Tied to one protocol             | `masque`                                    | Carries the MASQUE protocol only, so it cannot be used with VLESS.           |

- **Note:** `raw` used to be called `tcp`, and Xray still accepts the old name.

> TODO: check whether the client's link has to say `tcp` or `raw` for the client application to accept it.

##### 1.5.2 `rawSettings`

> `rawSettings`: both fields at their defaults, written out.

The settings that belong to the chosen `raw` method in 1.5.1. Xray reads this object only when `method` is `raw`. It has two fields, and both are off by default.

**Jerry sets:**

- `acceptProxyProtocol`: `false`.
- `header`: `{ "type": "none" }`.

Both are the defaults. Jerry writes them out anyway, purely as a way for me to not get confused. It's cosmetic tho, leaving it empty results in the default.

| Field                 | What it is                                                                                                                                                                  |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `acceptProxyProtocol` | Whether the inbound expects a PROXY protocol line at the start of every connection. A reverse proxy in front of Xray sends that line to pass on the visitor's real address. |
| `header`              | A fake header placed in front of the data. `none` sends the data as it is; `http` makes it look like an HTTP request and response.                                          |

- **Why `acceptProxyProtocol` is off:** nothing sits in front of Jerry's server, so no one would send that line. With the setting on, a connection that does not start with it is closed, which would lock out every client.
- **Why `header` is `none`:** the fake HTTP header is a disguise for a link that has no TLS. Under REALITY the link already looks like TLS to a real site, so a fake header would add nothing.

*FYI:*

- **Note:** `acceptProxyProtocol` applies to inbounds only.
- **Note:** with `header` set to `http`, both ends need the same fake header settings.

##### 1.5.3 `security`

> `security`: `reality`.

The transport security: the encryption and handshake layer that wraps the link. One kind is chosen, and both ends have to choose the same one. If you use Vless encryption and not in shit hole country that runs DPI, you can skip security.

**Jerry sets:** `reality`.

- **Why:** REALITY makes the link look like a TLS connection to a real, well-known website(the  `target`). It needs no domain name and no certificate of the server's own, and someone who probes the server is shown that real website. The details are in `realitySettings` (1.5.4).
- **The client's link** names the same kind.

*FYI:*

| Value     | What it means                                                                   | Works with             |
| --------- | ------------------------------------------------------------------------------- | ---------------------- |
| `none`    | No security layer. This is the default.                                         | -                      |
| `tls`     | Ordinary TLS, with a certificate of the server's own.                           | all eight methods      |
| `reality` | A modified TLS that borrows the look and the handshake traits of a target site. | `raw`, `xhttp`, `grpc` |

- **Note:** `hysteria` and `masque` require `tls`.
- **Note:** with VLESS, `none` is only acceptable on a trusted private network, or with VLESS Encryption turned on.

##### 1.5.4 `realitySettings`

Manual: [REALITY](https://xtls.github.io/en/config/transports/reality.html)

The settings that belong to the `reality` security. REALITY is a modified TLS: the server answers the client's hello the way a real website, the `target`, would answer it, so the link looks like a visit to that site.

The server checks every incoming connection. One that passes every check is treated as a real client. One that fails any of them is forwarded to the `target`, so the visitor gets the real website. Here a rough flow:

```text
1. The client sends its hello
     It is a TLS 1.3 hello, shaped like a browser's. -> (in uTLS finger print, configured below)
     Its server name field (SNI) carries the decoy's domain.
     Its session ID field, 32 bytes that are normally random, carries a hidden token:
       the client's Xray version, the current time, and its short ID.
     The token is locked with a secret that only the holder of `privateKey` can rebuild. -> (this is why REALITY does not need a certificate of its own, while HYS2 does)

2. The server checks the hello
     Is it TLS 1.3?
     Is the server name on `serverNames`?
     Does the token unlock?
     Is the version within `minClientVer` and `maxClientVer`?
     Is the time within `maxTimeDiff`?
     Is the short ID on `shortIds`?

3. The server answers
     every check passed  ->  it answers the way the decoy would, with a temporary certificate
                             stamped so the client can tell this is the real server. The link is up.
     any check failed    ->  it forwards the connection to `target`, and the visitor
                             gets the real website. Basically becoming an unpaid CDN / accelerator node (unless you throttle, look below)
```

The object has 18 fields: 12 for the server, covered here, and 6 for the client, which travel in the client's link.

```jsonc
{
  // Not valid JSON: the numbers are for reference only.
1.5.4         "realitySettings": {
1.5.4.1         "show": false,
1.5.4.2         "target": "<DECOY DOMAIN>:443",
1.5.4.3         "xver": 0,
1.5.4.4         "serverNames": ["<DECOY DOMAIN>"],
1.5.4.5         "privateKey": "<GENERATED: xray x25519, the private half>",
1.5.4.6         "minClientVer": "",
1.5.4.7         "maxClientVer": "",
1.5.4.8         "maxTimeDiff": 0,
1.5.4.9         "shortIds": ["<GENERATED: openssl rand -hex 8>"],
1.5.4.10        "mldsa65Seed": "",
1.5.4.11        "limitFallbackUpload": { "afterBytes": 0, "bytesPerSec": 0, "burstBytesPerSec": 0 },
1.5.4.12        "limitFallbackDownload": { "afterBytes": 0, "bytesPerSec": 0, "burstBytesPerSec": 0 }
              }
}
```

> TODO: decide how the decoy for `target` is chosen.

###### 1.5.4.1 `show`

> `show`: `false`.

A debug switch. When it is on, the server writes a line about every incoming connection to its log: who connected, and whether the connection passed REALITY's checks.

**Jerry sets:** `false`, the default. The output is only useful for studying who is knocking on the server, and it records visitors' addresses.

*FYI:*

- **What it prints when turned on:** the visitor's address and port; the client's Xray version, clock and short ID, if its token unlocked; whether the checks passed; and for a failed one, the server name that was forwarded to `target`.
- **Where:** the system journal, read with `journalctl -u xray`.
- You can set this to true , if you wanna see if your node actually gets probed or not. If you do find evidence of probing , let me know, because we might be over-paranoid here.

###### 1.5.4.2 `target`

> `target`: `<DECOY DOMAIN>:443`.

*Note on Naming:*
> The manual calls this target, used to be called dest. I call it Decoy sometimes because I find it cleaner. Don't get it twisted.

The real website REALITY imitates, written as a domain and a port, for example `example.com:443`. It is required. The server uses it in two ways:

- **For a connection that fails the checks:** the connection is forwarded to the target, so the visitor gets the real site.
- **For a connection that passes:** basically the server copies the target's homework; it asks the target how it would answer this hello, and answers its own client with the same choices.

**Jerry sets:** the decoy's domain, user input, with port `443`.

- **Choosing a decoy:** the manual's best practice is a site on the same network (ASN) as the server, so that the name and the server's address plausibly belong together. A site behind a CDN is a poor choice: the server would then relay traffic to that CDN for anyone who knocks.
- **What Jerry does about it:** nothing automatic. Jerry does not check the decoy against the server's network. It prints a warning about this choice and carries on.
- **A free benefit:** if the target supports the post-quantum key exchange, the link between client and server uses it too.

*FYI:*

- **Old name:** `dest`. Both names still work.
- **It marks the server side:** Xray treats a config that has this field as a server config. It must never appear in a client's config.
- **Comparing candidates:** `xray tls ping <domain>` shows how a site answers, including whether it supports the post-quantum key exchange.

###### 1.5.4.3 `xver`

> `xver`: `0`.

Whether the server adds a PROXY protocol line when it forwards a connection to the `target`. That line tells the target the visitor's real address. Without it, the target only sees the server's own address, because the server is the one opening the connection to it.

**Jerry sets:** `0`, the default, which means the line is not sent.

- **Why:** the line only makes sense for a target that expects it, which means a site the user runs and has set up to read it. Jerry's decoy is someone else's website, and an unexpected first line would break the connection to it.

*FYI:*

| Value      | What it means                                                   |
| ---------- | --------------------------------------------------------------- |
| `0`        | No line is sent.                                                |
| `1` or `2` | The line is sent, in that version of the PROXY protocol format. |

- **Note:** this is the same PROXY protocol as `acceptProxyProtocol` in 1.5.2, pointed the other way. There the server would read the line. Here it would write it.

###### 1.5.4.4 `serverNames`

> `serverNames`: `["<DECOY DOMAIN>"]`, one name.

The list of server names a client may put in its hello. It is required. If the name in a hello is not on this list, the connection fails and is forwarded to the `target`. This is the first check the server makes, before it looks at the token.

**Jerry sets:** one name, the decoy's own domain, the same one used in `target`.

- **Why one name:** the list is never visible from outside. It only decides which names Jerry's own client may use, and that client uses one.
- **The client's link** carries this name. It is what the client puts in its hello as the server name.

*FYI:*

- **Which names are valid:** names the target itself would accept, which in practice means the names on its certificate.
- **No wildcards:** `*` is not supported.
- **An empty string in the list:** the server then also accepts a hello that carries no server name at all, and the client sets its server name to an IP address. The target has to be a site that does not reject such a hello.

###### 1.5.4.5 `privateKey`

> `privateKey`: `<GENERATED: xray x25519, the private half>`.

The server's permanent X25519 private key. It is required. Its public half is handed to the client in the link, before any connection is made, which is why REALITY needs no certificate from a certificate authority.

**Jerry sets:** a key pair generated on the server with `xray x25519`. The private half goes here. The public half goes into the client's link.

- **How it is used:** the client already holds the public half, so it can combine it with a fresh key of its own before sending anything. The server combines the client's fresh key with this private key and arrives at the same secret.
- **What the secret does:** two jobs. It unlocks the token hidden in the client's hello, which is how the server recognizes its client. And it stamps the temporary certificate the server sends back, which is how the client recognizes its server.
- **Keep the link private:** anyone who holds the public half can pass this check.

*FYI:*

- **The public half can be rebuilt** from the private one with `xray x25519 -i <private key>`.
- **On the client's side** the public half is called `password`. It used to be called `publicKey`, and was renamed because it must not be made public.