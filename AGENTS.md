# polytail for agents

This machine reaches several Tailscale tailnets at once through `polytail`.
Each tailnet is a separate userspace `tailscaled` behind a loopback proxy. None
of them are on the system network stack, so a plain `ssh host` or
`curl http://host` will not reach them.

## Look before connecting

```sh
polytail status          # which tailnets exist and are Running; nonzero exit if any are not
polytail own <host>      # which tailnet a name belongs to; nonzero exit if none or ambiguous
polytail ping -n <net>   # is that tailnet actually passing traffic
```

The tailnet names and ports are in `~/.config/polytail/nets.conf`.

## Addressing rules

- Use `host.<net>` (e.g. `nas.home`) or the full MagicDNS name
  (`nas.tailxxxx.ts.net`). Never use a bare short name: the same hostname often
  exists on more than one tailnet.
- Never use a `100.x.y.z` or `fd7a:115c:a1e0::` address without `-n <net>`.
  Those ranges are reused by every tailnet, so an address alone does not say
  which machine it is.
- If `polytail` says a name matches several tailnets, or no tailnet owns it,
  stop and ask which one is meant. Do not try the other tailnets to see which
  one works.

## Reaching things

```sh
ssh nas.home                                             # if ~/.ssh/config includes polytail's ssh_config
ssh -o ProxyCommand='polytail connect %h %p' nas.home    # if it does not
polytail run -n home -- curl http://nas:8080/            # any proxy-aware command; short names resolve inside that tailnet
polytail connect nas.home 5432                           # raw TCP on stdin/stdout
```

`polytail run` is scoped to the one command. There is no "current tailnet" to
switch to, and `eval "$(polytail env -n <net>)"` affects only that shell.

## Setting up and keeping it running

Only do this when asked: these commands change state for every program on the
machine.

Add a tailnet by appending a line to `~/.config/polytail/nets.conf`:

```
# <name> <socks-port> <magicdns-suffix> [canary host:port]
home     1055  -  nas:22
```

The name is lowercase `[a-z0-9-]`. Each tailnet uses its SOCKS port and the
port after it, so space them out (1055, 1065, 1075, ...). Leave the suffix as
`-`; `polytail up` fills it in after login. The canary is optional.

```sh
polytail validate        # port and suffix conflicts, missing binaries
polytail up home         # starts the daemon; first time, prints a login URL
polytail install home    # LaunchAgent: starts at login, restarts if it dies (macOS only)
polytail status          # confirm Running
```

The login URL must be opened by the human, signed in as themselves. Don't
substitute an auth key. After login the machine appears in that tailnet as
`<hostname>-<net>`.

Without `install` the daemon dies on reboot. With it, launchd keeps it up and
`polytail up` just restarts it. To make `ssh host.<net>` work without a
`ProxyCommand`, generate the ssh config once and include it:

```sh
polytail ssh-config > ~/.config/polytail/ssh_config
grep -q 'polytail/ssh_config' ~/.ssh/config || echo "Include ~/.config/polytail/ssh_config" >> ~/.ssh/config
```

`Include` has to come before any `Host` block in `~/.ssh/config`, so check
where the line landed. Re-run `ssh-config` after adding a tailnet.

Other lifecycle commands:

```sh
polytail down home       # disconnect, daemon stays resident, reconnect needs no login
polytail stop home       # kill the daemon
polytail uninstall home  # remove the LaunchAgent (also stops it)
```

Logs are in `~/Library/Logs/polytail/<net>.err.log`. Periodic
`handshakeZeroed` lines there are normal; judge health with `polytail ping`.

## Limits

- TCP through a proxy only: no UDP, ICMP, subnet routes, or programs that
  ignore `ALL_PROXY`/`HTTP_PROXY`. `ping` will not work; use `polytail ping`.
- The plain `tailscale` CLI talks to the GUI client's tailnet, not the
  `polytail` ones. Use `polytail` for these tailnets.
- `polytail ping` reports Tailscale's path probe plus a TCP dial to the canary.
  `tcp/22 ok` means the port accepted a connection, not that the service is
  healthy.
