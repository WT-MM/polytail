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

## Limits

- TCP through a proxy only: no UDP, ICMP, subnet routes, or programs that
  ignore `ALL_PROXY`/`HTTP_PROXY`. `ping` will not work; use `polytail ping`.
- The plain `tailscale` CLI talks to the GUI client's tailnet, not the
  `polytail` ones. Use `polytail` for these tailnets.
- `polytail up` may print a login URL that a human has to open in a browser.
  Don't run `up`, `down`, `stop`, `install` or `uninstall` unless asked; they
  change state for every program on the machine.
- `polytail ping` reports Tailscale's path probe plus a TCP dial to the canary.
  `tcp/22 ok` means the port accepted a connection, not that the service is
  healthy.
