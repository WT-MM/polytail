# polytail

Use several Tailscale tailnets at once, from one machine.

The official client holds many profiles but only one can be active. Switching is
global: it drops every connection through the old tailnet and rewrites your
routes and DNS. If you have more than one tailnet (your own, one a friend or
family member shares with you, another for a homelab or side project), you
switch constantly and you are never sure which one a command just went to.

`polytail` runs one userspace `tailscaled` per tailnet and routes by name
ownership, so all of them stay reachable at the same time.

```
$ polytail status
TAILNET      STATE     TAILNET-ID   NODE            ADDRESS         PEERS   EXPIRY  PROXY
system(gui)  Running   tailaaaaaa   laptop          100.73.81.74    18/34   84d     system utun
home         Running   tailbbbbbb   laptop-home     100.67.71.52    17/48   180d    socks5://127.0.0.1:1055
lab          Running   tailcccccc   laptop-lab      100.94.140.7    26/40   180d    socks5://127.0.0.1:1065

$ polytail ping
TAILNET    TARGET          RTT     PATH                        SERVICE
home       nas             6ms     direct 192.168.1.244:41641   tcp/22 ok
lab        gateway         48ms    DERP(nyc)                    tcp/22 ok
```

## Why names, not addresses

Tailscale allocates `100.64.0.0/10` per tailnet, so the same address means
different machines on different tailnets. Every tailnet shares the identical
`fd7a:115c:a1e0::/48` IPv6 prefix. `100.100.100.100` is the MagicDNS resolver in
all of them, and the system resolver can only ever serve one. Short hostnames
collide freely: two tailnets can easily share a dozen, like two laptops called
`macbook` and two hosts called `influxdb`.

So there is nothing in an address or a bare hostname that identifies a tailnet.
`polytail` gives each tailnet ownership of its MagicDNS suffix and a synthetic
`.<name>` alias, and resolves inside the owning daemon:

```
$ polytail own nas.home
home  nas.tailbbbbbb.ts.net

$ polytail own influxdb
no tailnet owns influxdb

$ polytail own 100.64.1.1
polytail: 100.64.1.1 is a raw address and 100.64/10 is per-tailnet; pass -n <net>

$ polytail own 100.100.100.100
polytail: 100.100.100.100 is the resolver in every tailnet
```

It fails closed, and the rules are worth stating exactly:

- Real MagicDNS suffixes are checked before synthetic `.<name>` aliases.
- A name matching two tailnets is refused, not resolved to the first.
- An unrecognised `*.ts.net` name is refused rather than reinterpreted as an
  alias.
- An explicit qualifier beats `-n`: `connect -n home db.lab 22` is an
  error, not a silent connection to `home`'s `db`.
- Raw addresses need `-n`, and `100.100.100.100` is refused outright.

A name resolves in exactly one tailnet or in none. There is never a fallback to
a second tailnet, because that is how you land on the wrong machine.

## Coexistence

Each tailnet runs as `tailscaled --tun=userspace-networking`, which creates no
`utun`, installs no routes, and does not touch system DNS. Verify it yourself:
`ifconfig` shows one Tailscale interface, `netstat -rn` shows every `100.64/10`
and `fd7a:115c:a1e0::/48` route pointing at it, and `scutil --dns` shows one
tailnet's search domain.

Two consequences:

- The official GUI client keeps working, on whichever tailnet you like. It owns
  the system layer alone.
- Another VPN cannot fight `polytail`, because the fight is over the TUN device,
  the routing table, and the resolver, and `polytail` claims none of them. You
  can quit Tailscale entirely and the `polytail` tailnets stay up.

One coupling remains: an exit node on the GUI client captures the system default
route, so `polytail` traffic rides through it too.

## Install

Requires `bash`, `jq`, `python3` (for `pac serve`), and Tailscale's open-source
`tailscale`/`tailscaled` binaries — on macOS, `brew install tailscale`, which is
separate from the App Store client.

```sh
git clone https://github.com/WT-MM/polytail
ln -s "$PWD/polytail/polytail" ~/.local/bin/polytail   # or anywhere on PATH
```

A symlink means `git pull` takes effect with no reinstall. Note that
`/usr/local/bin/tailscale` may be the App Store client's launcher rather than
the open-source CLI, so `polytail` defaults to `/opt/homebrew/bin/tailscale`.
Override with `POLYTAIL_TS` and `POLYTAIL_TSD` if yours live elsewhere.

## Setup

A tailnet needs a name, a loopback SOCKS port, and its MagicDNS suffix. Declare
it in `~/.config/polytail/nets.conf`:

```
# <name> <socks-port> <magicdns-suffix> [canary host:port]
home     1055  tailbbbbbb.ts.net  nas:22
lab      1065  tailcccccc.ts.net  gateway:22
```

The HTTP proxy is the SOCKS port plus one. Leave the suffix as `-` and
`polytail up` fills it in after the first login.

```sh
polytail up          # starts each daemon, prints a login URL per tailnet
polytail install     # LaunchAgents, so they survive a reboot
polytail validate    # suffix and port conflicts
```

Log in through the browser as yourself. Do not use a shared or tagged auth key:
a tag replaces your identity, and Tailscale SSH and your ACLs stop seeing you.
Each tailnet gains one extra machine entry named `<host>-<net>`.

The canary is a `host:port` that should always answer. `polytail ping` dials it
to distinguish "the tailnet is up" from "the thing I need is reachable".

## Commands

```
status                    state, node, peers, key expiry, proxy per tailnet
ping [-n net] [host]      DISCO path + canary TCP dial, every tailnet at once
own <host>                which tailnet owns a name; fails closed
validate                  suffix and port conflicts, missing binaries

connect [-n net] <h> <p>  raw TCP through the owning tailnet (ProxyCommand)
run -n <net> -- <cmd>     run a command with that tailnet's proxies
env -n <net>              print proxy environment variables
ssh-config                ssh stanzas plus per-tailnet known_hosts
pac [serve]               proxy auto-config so browsers route by hostname

up|down|stop [net...]     down keeps the daemon resident, stop kills it
install|uninstall [net..] LaunchAgent for reboot persistence
```

`down` disconnects but leaves the daemon resident with its credentials loaded,
so reconnecting is instant and needs no browser round trip. `stop` ends the
process and releases the ports.

### SSH

```sh
polytail ssh-config >> /dev/stdout   # inspect, then:
echo "Include ~/.config/polytail/ssh_config" >> ~/.ssh/config
ssh nas.home
```

Each tailnet gets its own `known_hosts`, so two machines sharing a hostname stay
distinct instead of looking like a MITM. Short names are deliberately absent
from the generated config.

### Other programs

```sh
polytail run -n home -- curl http://influxdb:8086/ping
eval "$(polytail env -n home)"
polytail pac serve      # then point the browser at http://127.0.0.1:1099/proxy.pac
```

The generated PAC routes by real MagicDNS suffix only, never by the synthetic
alias, since an alias is not a DNS name and a tailnet named after a real TLD
would otherwise capture public traffic. Addresses in `100.64/10`,
`fd7a:115c:a1e0::/48`, and `100.100.100.100` are sent to a dead proxy rather
than `DIRECT`, because going direct would leak them to whichever tailnet the
system client is on.

`run` and `env` are scoped to one command or one shell. There is deliberately no
persistent "current tailnet": hidden state is the ambiguity this tool exists to
remove.

## Logs

Under a LaunchAgent, `tailscaled` writes to stderr, so the interesting file is
`~/Library/Logs/polytail/<net>.err.log`; the `.log` beside it stays empty.
Expect roughly 2 MB per tailnet per day, mostly `RAW-STDERR` lines where
`tailscaled` echoes its own logtail JSON. Nothing rotates these; that is left
to `newsyslog` or your own housekeeping.

`wg: Failed to derive keypair: invalid state for keypair derivation:
handshakeZeroed` appears periodically and is normal: a handshake expired while a
packet was still queued for an idle peer. Judge health with `polytail ping`, not
by grepping logs.

## What this does not do

Userspace networking is a proxy, not a tunnel. Only proxy-aware and
`ProxyCommand`-aware programs reach these tailnets: SSH, `scp`, `rsync`,
`curl`, git, and anything that honours `ALL_PROXY`. It does not carry UDP,
multicast, ping, subnet routes, or programs that ignore proxy settings. Keep
those on the GUI client.

`ping` reports Tailscale's DISCO probe, not ICMP. A `DERP(...)` path is a pass;
DERP is a valid route, just a relayed one. `tcp/22 ok` means a TCP connection
was accepted at that instant, not that the service behind it is healthy.

macOS is what this is tested on. The daemon flags and `ssh_config` generation
are portable to Linux, but `install` writes LaunchAgents and refuses to run
elsewhere; a systemd user unit is not written yet.

## Alternatives

- Tailscale's own fast user switching, if one tailnet at a time is enough.
- Machine sharing, which exposes individual hosts into a tailnet you are already
  on and needs no extra daemon. Better when it fits; it is per-user, strips tags
  and subnet routes, and needs the owning tailnet's cooperation.
- On Linux, one `tailscaled` per network namespace gives transparent routing
  instead of a proxy, at the cost of hand-maintained veth pairs and no MagicDNS.
- [TailMux](https://tailmux.app) is a paid closed-source tool covering similar
  ground with an embedded `tsnet` backend. Its command surface suggested several
  features here.

## License

MIT
