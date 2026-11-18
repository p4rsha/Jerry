# Jerry

Jerry takes a fresh Debian or Ubuntu server to a working Xray-core based VPN as fast as possible.

## Why

VPN servers that are not sitting behind another proxy node have a tendency to not last long. Sooner or later the IP gets hit:

- **By a censor.** Deep packet inspection (DPI) spots the server, and the IP gets blocked.
- **By a streaming service.** The IP tends to work at first, then gets flagged after a while.
- **By Google.** Don't ask me how, they somehow can tell you are using a VPN after some time (the joy of being under OFAC sanctions).

Jerry's answer is agility. Rather than going full Solid Snake and trying to out-engineer a team full of people that get paid to block you, just spin up a new server, run Jerry, and carry on in minutes. A middle finger at its finest.

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
