<img src="logo-wide.svg" alt="byway" width="220">

<sub>A cairn — the stack of stones that marks a path where none is visible.</sub>

# byway

**Split tunnelling for an OpenWrt router.** Part of your traffic goes through
your VPN, the rest goes direct. The engine is
[Xray](https://github.com/XTLS/Xray-core).

![OpenWrt 22.03+](https://img.shields.io/badge/OpenWrt-22.03%2B-00B5E2)
![engine Xray](https://img.shields.io/badge/engine-Xray--core-333)
![IPv4 only](https://img.shields.io/badge/IP-IPv4%20only-orange)
![GPL-2.0](https://img.shields.io/badge/license-GPL--2.0-blue)

*[Русская версия](README.md)*

---

> ### ⚠️ Read this before installing
>
> **Version 0.0.0 — the first public release.** byway runs every day on one
> router: 1500 domains, 300 subnets, and a family that notices breakage
> immediately. But still just **one** — the author had no other hardware.
>
> | | |
> |---|---|
> | fully verified | Cudy WR3000S v1 (MT7981, aarch64), OpenWrt 25.12.5 |
> | verified on a test bench | install and removal on 22.03–25.12, both package branches |
> | never verified at all | IPv6, a foreign architecture in production, behaviour under load |
>
> **The code was written by an AI** — Claude, to a human's brief and
> corrections. That is said up front rather than in a footnote:
> [what was done about it](#written-with-an-ai).
>
> **A report that it did not work for you is worth more than any review.** A
> failure report beats a success one:
> [issues](https://github.com/Tomonj1/byway/issues).

---

## Contents

[Why this exists](#why-this-exists) · [Requirements](#requirements) · [What
byway does not do](#what-byway-does-not-do) · [Installing](#installing) ·
[First run](#first-run) · [What it can do](#what-it-can-do) · [How it
works](#how-it-works) · [The web UI](#the-web-ui) · [Commands](#commands) ·
[Updating](#updating) · [Engine version](#engine-version) ·
[Removal](#removal) · [Compatibility](#compatibility) · [Written with an
AI](#written-with-an-ai)

---

## Why this exists

byway gives you access to nothing on its own: the VPN is yours, and what opens
through it depends on the VPN, not on byway. byway's job is different — to
**decide what goes through the VPN and what goes around it**, and to do it on
the router, so you don't have to run a client on every device in the house.

What usually has to bypass the tunnel is local services: banks, government
portals and marketplaces often refuse foreign addresses and simply stop working
over a VPN.

byway solves **one** problem and does not build a second life around it. There
is no traffic-graph screen, no private vocabulary, no subscription to someone
else's rule sets you have to learn separately. There are lists as plain files, a
key as a link, and five tabs in the web UI.

If you want the full toolbox, there are
[PassWall](https://github.com/xiaorouji/openwrt-passwall) and
[OpenClash](https://github.com/vernesong/OpenClash): they do more and weigh
accordingly.

**What it is, technically.** Neither a package nor a binary: a set of POSIX sh
scripts plus a LuCI panel. Xray carries the traffic — byway decides what goes
where, builds the engine's config, installs the kernel rules and watches that
none of it falls apart. Plumbing, in other words.

---

## Requirements

| | |
|---|---|
| **OpenWrt 22.03 or newer** | a hard boundary, see below |
| **`kmod-nft-tproxy`, `kmod-nft-socket`** | the installer fetches them |
| **`curl`** | the installer fetches it |
| **Flash space** | byway itself is under a megabyte; the Xray engine needs ~18 MB |

**Why 22.03 is a boundary and not a preference.** From 22.03 the firewall is
firewall4 on nftables, and byway stands entirely on it. On 21.02 and older it is
firewall3 with iptables — a different mechanism, a mark-based rule will not go
in, and neither guests nor zones with an `input REJECT` policy would get the
tunnel. On such a system the installer **refuses to run** and says why: half a
working byway is worse than an honest refusal.

The lower bound comes from the engine: Xray is already in the feed in 22.03, and
byway can fetch it from GitHub itself. For
[podkop](https://github.com/itdoginfo/podkop) that bound is one branch higher
only because sing-box appears in the feeds from 23.05. sing-box support is
planned for byway too — the plumbing does not depend on the engine — and then
this difference disappears.

The package manager is detected automatically: `apk` from 25.12, `opkg` on 24.10
and older.

**About the 25 MB for a GitHub install.** Eighteen for the engine itself plus
room to unpack; the archive stays in memory, not on flash. With less, the
installer does not ask — it takes the firmware feed's version and says so out
loud.

---

## What byway does not do

Worth knowing before installing, not after.

- **IPv4 only.** With IPv6 up, some connections will bypass byway.

  > **IPv6 is planned — and it needs a person who has IPv6.** The reason it is
  > missing is not laziness: the ISP byway was written on does not provide it, so
  > there was nowhere to verify interception. Rules written blind
  > are **worse than no rules** in an interception path — they quietly send part
  > of the traffic the wrong way, and it can take weeks to notice. If you have
  > IPv6 and are willing to test on your own router,
  > [say so](https://github.com/Tomonj1/byway/issues).

- **No `hysteria2`, `tuic`, `wireguard`** — they are not in Xray-core.
- **One engine.** Xray for now. byway's plumbing does not depend on the engine,
  so **a choice of engine at install time is planned** — sing-box first — but not
  today.
- **A client with its own DNS bypasses it.** DNS decides the route: a device
  with Private DNS or DoH in the browser asks someone other than the router and
  gets the real address rather than the placeholder. `byway doctor` says so. The
  cure is either to turn encrypted DNS off on the device, or to add the
  service's **subnets** alongside its domains: a subnet works by address, and so
  works for whoever asked someone else for it. One does not replace the other —
  subnets complement domains.

---

## Installing

One line:

```sh
sh -c "$(wget -O - https://raw.githubusercontent.com/Tomonj1/byway/v0.0.0/install.sh)"
```

If `raw.githubusercontent.com` is unreachable, the same through a mirror. **The
mirror is someone else's** — the public `gh-proxy`, the same one
[Zapret-Manager](https://github.com/StressOzz/Zapret-Manager) uses. We neither
run it nor check what it serves. If you would rather not trust a third party
inside a root install, take the archive the third way and read it first.

```sh
wget -T 10 -O /tmp/byway-install.sh \
  "https://v4.gh-proxy.org/raw.githubusercontent.com/Tomonj1/byway/v0.0.0/install.sh" \
  && sh /tmp/byway-install.sh
```

As an archive, if you want to read it first:

```sh
cd /tmp
wget -O byway.tar.gz https://github.com/Tomonj1/byway/archive/refs/tags/v0.0.0.tar.gz
tar xzf byway.tar.gz && cd byway-0.0.0
sh install.sh
```

**How to check the installer rather than trust it.** It runs as root — it could
not touch the network otherwise:

- the link points at a **tag**, not a branch: you install what is marked with a
  version, not what the author pushed a minute ago;
- it is one readable file: `wget -O - …` without `| sh` shows all of it;
- it names every step it takes out loud and does nothing silently.

**The first two ways download twice.** The one-liner puts only `install.sh` on
the router; it fetches the rest of the package with a second request, from the
same tag that is baked into it as a constant. If the tag does not exist it says
out loud that it took the `main` branch, rather than pretending it installed a
tagged version. The third way goes to the network once, entirely in front of you.

**It asks about what is optional:** where to get the Xray engine and which
version, and whether `base64` is needed (only for `vmess://` and `ss://` keys).
Mandatory pieces are installed without questions. With no terminal it takes the
defaults and says so.

**The installer does not touch your configuration or lists** — which is why
running it again is safe, and why updates install the same way.

---

## First run

**1. The key.** Web UI: *Services → Byway → Main*, the "Key" field — the whole link
from your VPN: `vless://`, `vmess://`, `trojan://`, `ss://` or `socks://`. Or in
the console:

```sh
uci set byway.main.node_url='vless://…'   # or vmess://, trojan://, ss://, socks://
uci set byway.main.enabled=1
uci commit byway
```

**2. The list.** The "Routes" tab or the file `/etc/byway/domains.lst`, one
entry per line; an entry covers subdomains too. Ready-made lists are attached
next to it with a checkbox. Or the "Everything through the VPN" mode, if there
is nothing to split.

**3. Start:**

```sh
/etc/init.d/byway enable
/etc/init.d/byway start
```

**4. Check** — `byway health`. The engine takes about fifteen seconds to come
up; there is nothing to check before that.

If something is wrong — `byway doctor`: it checks the environment and names a
cure for every fault.

---

## What it can do

### Routing

- **By domains and subnets.** Lists are plain files, one entry per line; an
  ordinary entry covers subdomains. If you need to be more precise, byway
  accepts Xray's forms: `full:` (that name only), `keyword:` (a match on a
  fragment), `regexp:`. `geosite:` and `ext:` are **not yet** accepted: they need a
  geodata file of a dozen megabytes, and the router has forty in total. If it
  matters to you more than the free space —
  [say so](https://github.com/Tomonj1/byway/issues), it is not hard to add. Non-Latin domains go in punycode. A line that does
  not parse is named out loud at build time and left out of the list.
- **Two modes.** "By lists" — only what is listed goes through the VPN.
  "Everything through the VPN" — all traffic, with a separate checkbox that
  keeps `.ru`, `.su` and `.рф` domains direct.
- **Several exits.** A separate list can be sent to a separate VPN: "these
  domains go there, everything else to the main one".
- **Ready-made lists** are attached with a checkbox and refresh either on a
  button or on their own — the interval is written in words: `12h`, `2h37m`,
  `1d`. Besides domains they bring subnets: services that work by address rather
  than by name need them. They do not argue with your own lists; entries are
  merged.

  **byway measures their width and says it out loud:** whole hosting ranges pull
  other people's traffic into the tunnel, and it is better to know that as a
  number in advance than as lost speed later.
- **DNS** can stay direct or go through the VPN. Domains from the list are not
  affected: a built-in resolver answers those locally.

### Connection

- **Keys:** `vless`, `vmess`, `trojan`, `shadowsocks`, `socks`.
  **Transports:** `tcp/raw`, `ws`, `grpc`, `httpupgrade`, `xhttp`.
  **Security:** `tls`, `reality`.
- **Several keys at once:** pick one by hand or let Xray do it — it measures
  latency and routes through the fastest live one.
- **Subscription:** fetch a list of keys by URL and pick one.
- **Your own outbound config** — for what byway does not parse from a link.
- **Multiplexing** — several client streams inside one connection to the VPN.
  It is a web-transport technique; byway has four of those: `ws`,
  `httpupgrade`, `xhttp` and `grpc`.

  It is enabled where the transport **does not multiplex itself** — that is, on
  `ws` and `httpupgrade`. `xhttp` has its own `xmux` for that, `grpc` has
  `multiMode`, and a second layer on top only gets in the way; with
  `xtls-rprx-vision` byway leaves it off too — Vision splits the stream itself.
  It says so out loud in every such case rather than staying quiet.

  Measured on `ws`: session setup went **588 → 149 ms**, and connections to the
  server dropped from about fifty to exactly eight.

### When something goes wrong

- **A plumbing watchdog.** Every five minutes it checks that the rules are in
  place while the engine is running, and puts them back if something outside
  removed them — someone else's `nft flush ruleset`, a firewall4 update, a
  neighbouring service. It leaves deliberately removed plumbing alone.
- **Failure behaviour** is a choice — see [How it works](#how-it-works).
- **Checks:** `byway doctor` for the environment, `byway health` for whether it
  works right now, `byway probe` to test a key in isolation without touching the
  working tunnel.
- **A state log** — what changed and when: drops, restarts, plumbing removed.
  Written every five minutes and only when there is something to write.
- **A report for a bug thread** — `byway report`: state, environment and
  diagnostics as one piece of text, **without the VPN key**.

### Control

- **The LuCI web UI**, from which everything is done: key, mode, lists,
  diagnostics. You never have to touch the console.
- **A console menu** — `byway menu`, the same actions.
- **Export and import.** All settings and lists as one piece of text:
  `byway export` and `byway import`. The export comes with or without the key
  (`--no-key`).
- **Usage statistics** (off by default): which list entries are actually used.
  Useful when deciding what to remove. Everything stays on the router.

---

## How it works

A domain from the list gets an address from byway that does not exist on the
internet (from the `198.18.0.0/15` range). After that it is simple: any packet
to such an address is by definition the traffic that has to be diverted, and
that is visible without looking inside.

```
dnsmasq → Xray DNS inbound → placeholder address for a listed domain
                                    ↓
                  nft rules divert that traffic
                                    ↓
              Xray restores the domain and decides where to send it
```

Where things go:

| | |
|---|---|
| `byway` — one script, all the logic | `/usr/local/bin/byway` |
| the procd service | `/etc/init.d/byway` |
| settings | `/etc/config/byway` |
| lists, engine config, logs | `/etc/byway/` |
| the panel | `/www/luci-static/resources/byway/` and `.../view/byway/` |

**If Xray did not come up, interception is not enabled either.** The house is
left with the internet and without the tunnel, rather than without DNS — that is
a deliberate choice.

This is easy to get wrong, so plainly: **without the tunnel the list does not
stop working — it starts working AROUND the VPN.** The domains resolve to real
addresses, connections open as usual, and from your home address. Sites open,
everything looks intact, and there is no protection — with nowhere to learn that.

That is what the second failure model — **"do not let it through"** — undoes.
What exactly it closes depends on the list mode, and the difference is large:

| mode | what stays closed until the VPN returns |
|---|---|
| by lists | the list only; the rest of the internet works |
| everything through the VPN | the whole way out — this is the full kill switch |

Access to the router itself (LuCI, ssh) is untouched in either case. The switch
is on the "Main" tab.

---

## The web UI

*Services → Byway*, five tabs:

| tab | what is there |
|---|---|
| **Main** | whether it works, through what, and how to change that: state, key, connection mode, failure behaviour |
| **Routes** | what goes through the VPN: mode, your lists, ready-made lists, directions |
| **Network** | whose traffic to divert, DNS, interception ports and addresses |
| **Maintenance** | statistics, state log, settings transfer, updates, full state |
| **Advanced** | values you change once in a lifetime |

> ⚠️ **Clear the browser cache after updating byway.** LuCI appends the version
> of **LuCI itself** to a module's URL, not the file's, so a byway update does
> not move the cache: the browser keeps showing the old tab, and there is no way
> to tell by looking. Ctrl+F5 helps, but only re-fetches **the modules of the
> open page** — you would have to do it on every tab. More reliable: F12 →
> Network → "Disable cache" → F5, without closing the tools.

---

## Commands

| command | what it does |
|---|---|
| `byway` | state and the list of commands |
| `byway menu` | console menu |
| `byway status` | what is working right now |
| `byway health` | service, VPN link, traffic, DNS |
| `byway doctor` | environment: modules, tools, space, conflicts |
| `byway gen` | rebuild the config from settings and lists |
| `byway plumb on\|off` | raise or remove interception |
| `byway check [LINK]` | parse a key and verify the config, no connections |
| `byway probe [LINK]` | test a VPN in isolation without touching the working tunnel |
| `byway sub URL` | fetch a subscription and show the keys |
| `byway presets` | refresh the ready-made lists |
| `byway top [N]` | what is actually used |
| `byway update [--check]` | see whether a new version exists, and install it |
| `byway report [file]` | a report for a bug thread: state and diagnostics, no key |
| `byway export [file]` | export settings; `--no-key` leaves the VPN key out |
| `byway import FILE` | apply settings from an export |
| `byway clear log\|stat\|all` | clear the state log, the statistics, or both |
| `byway show` | show the built config, without the key |
| `byway nft` | show the interception rules without applying anything |
| `byway version` | version |

---

## Updating

```sh
byway update --check     # see whether a new version exists
byway update             # install it
```

An update does not touch settings or lists. Clear the browser cache afterwards —
see the warning above.

**Checking for a version and installing one are different things, and they leave
different traces.**

| | default | what it does |
|---|---|---|
| `update_check` | **on** | asks GitHub once a day whether a newer release exists |
| `auto_update` | off | installs what it found on its own, at 04:00–05:00 |

A regular request from your home address is a steady "byway is installed here"
trace, readable at the ISP without any traffic inspection. It is turned off with
a checkbox on the "Maintenance" tab or `option update_check '0'`.

**The same is true of refreshing the ready-made lists** (`lists_update`) — it
also goes to GitHub on a schedule. The difference is the default: the version
check is on, the list refresh is **off**, and you set the interval yourself. Both
first try to go through the tunnel and only fall back to going direct — so the
trace is left exactly when the VPN is down.

**Auto-update** (`auto_update`) is off deliberately: it restarts the service,
which takes the tunnel away from the whole house. By turning it on you accept
that this happens at 04:00–05:00 **by the router's clock**. The time is **not
configurable**: the hour is hard-coded. A stock firmware sits in UTC, in which
case that is the morning — `byway doctor` will tell you when it lands for you.

A release is installed no sooner than three days after it appears (important ones
immediately) and only within the same minor version. If the tunnel does not come
up within two and a half minutes, byway puts the previous version back on its
own.

---

## Engine version

byway is not tied to a version of Xray: if no path to the engine is set, the one
from the package is used. The installer **asks** where to get the engine: from
the firmware feed (the default — the version OpenWrt built), from GitHub — there you
pick: `latest` is the default, `tested` is the one byway was verified on end to
end, or any version number; or nowhere, if you will point at a path yourself
later.

⚠️ **"Newest" and "stable" are different things for Xray.** XTLS marks
everything newer than `26.3.27` as a pre-release, so `latest` gives exactly that
one — stable, but noticeably behind. `tested` gives the one byway was verified
on end to end; that is a pre-release, and byway says so during installation. It
is what runs on the developer's router.

**It is worth keeping the engine fresh.** Xray moves fast: transports get fixed
and added. If the feed's version is old, put the binary next to it by hand and
point at the path.

**If the config stopped building after an engine update** — byway verifies every
build with the engine itself, so an incompatibility does not pass silently: the
config is simply not replaced and the previous one keeps working. That already
happened when 26.7 removed the `kcp`, `h2` and `quic` transports. What to do:

```sh
V=26.7.28                       # the version that worked
cd /usr/local/bin
wget -O xray.zip "https://github.com/XTLS/Xray-core/releases/download/v$V/Xray-linux-arm64-v8a.zip"
unzip -o xray.zip xray && mv xray "xray-$V" && chmod 755 "xray-$V"
uci set byway.main.xray_bin="/usr/local/bin/xray-$V" && uci commit byway
/etc/init.d/byway restart
```

Pick the archive for your architecture (`arm64-v8a`, `mips`, `mipsle` and so on
— see the release's file list). **Two engines do not always fit side by side:**
the binary is about 18 MB on flash, so remove the old one as soon as the new one
works.

And [tell us about it](https://github.com/Tomonj1/byway/issues): if the engine
changed what byway generates, that is fixed in byway rather than worked around
by every user separately.

---

## Removal

```sh
sh uninstall.sh              # settings and lists stay
sh uninstall.sh --purge      # remove everything, including the key
DRY_RUN=1 sh uninstall.sh    # show what would be done, change nothing
```

The script returns the network to its original state on its own: DNS goes back
to the ISP, rules are removed, the service is unregistered. It does not touch
the Xray engine or the network settings.

---

## Compatibility

**On real hardware:** Cudy WR3000S v1 (MediaTek MT7981, aarch64), OpenWrt
25.12.5. Developed and used daily: 1500 domains and 300 subnets, transports ws,
xhttp, httpupgrade and tcp+reality. This is the only combination where byway is
verified end to end — with a live tunnel, real traffic and real flash limits.

**On a test bench** (qemu, x86-64, `generic-ext4-combined` images):

| version | what was verified |
|---|---|
| 22.03.7 | full install, opkg branch |
| 23.05.6 | install; removal, dry run and `--purge` |
| 24.10.8 | install, engine from GitHub, opkg branch; removal and `--purge` |
| 25.12.5 | install, apk branch; settings import, clearing, list downloads |

"Removal verified" here is meant literally: a snapshot of the system is taken
BEFORE the install and AFTER `uninstall.sh --purge`, and they match byte for
byte — including dnsmasq and firewall settings, cron jobs, nft tables, routing
rules and the firmware keep list. The dry run is separately verified to change
not a single byte.

**What the bench does not verify, and it matters.** It is x86-64, so the choice
of engine build for your architecture is never executed there — and that is
exactly where a bug already lived (on MIPS the build with the wrong byte order
was downloaded). Flash in a VM is elastic, so the 43.7 MB ceiling is not
reproduced. The tunnel does not come up on the bench at all: install and cleanup
are verified, not operation.

There was no other hardware. The list of devices byway has been run on lives in
the [compatibility
reports](https://github.com/Tomonj1/byway/issues?q=label%3A%D1%81%D0%BE%D0%B2%D0%BC%D0%B5%D1%81%D1%82%D0%B8%D0%BC%D0%BE%D1%81%D1%82%D1%8C).
If you ran it, add yours: that is the single most useful thing you can report
right now — and a failure report beats a success one.

---

## Written with an AI

byway's code was written by Claude, to a human's brief and corrections: the link
parser, the nft rules and the web UI alike. That proves nothing by itself — so
here is how the generated was told apart from the verified.

**Reviews by independent agents, angle by angle.** Each agent got its own angle —
data from outside, permissions and secrets, shell mistakes, failure behaviour,
cleaning up after itself, races between consumers, neighbours on the router,
limits and volumes, clocks, promises against behaviour. Every finding was then
checked by a separate sceptic given the **opposite** task: to refute it, not to
confirm it.

There have been three reviews, all closed: the first with 106 findings, a
separate one for the web UI with 34, and the second, in four passes, with 98.

**What reading does not catch — and what was done about it.** Two findings were
missed by all four passes of the second review: `chain fwd` was rejected by the
kernel along with the entire table (`fwd` is a reserved word in nft), the subnet
side of the kill switch had never loaded since it was written, and the error was
muffled by `2>/dev/null || true`. What caught them was not a reader but a parser
— `nft -c`. Hence three benches that run **every** branch of what byway hands to
other programs: kernel rules through `nft -c`, the engine config through Xray
itself, the blocking snippet through `dnsmasq --test`, and UCI edits and cron
jobs against a stand-in configuration.

**Every config build is verified by the engine itself** — `xray run -test`. If
it is not accepted, the working config is not replaced and the tunnel keeps
running on the previous one.

**It runs every day on a live router.** Fifteen hundred domains, three hundred
subnets, and a family that notices immediately when something breaks.

**What none of this means.** byway has seen one router model and one OpenWrt
version, IPv6 was never tested at all, and full removal has been verified on a
live router exactly once.

---

## License

[GPL-2.0](LICENSE) — the same one OpenWrt itself lives under.
