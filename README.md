# wireguardian

**Don't fire a single packet until you *know* you're behind the VPN.**

A small, dependency-light opsec guard for WireGuard. It brings your tunnel up,
**proves your traffic is actually being tunnelled**, checks for DNS and IPv6
leaks, and can fail traffic *closed* — so the classic mistake of scanning while
you only *think* you're anonymous simply can't happen.

Shared as a note on VPN opsec hygiene. If it saves one person a leaked source
IP, it did its job.

## The problem it solves

`wg-quick up` reports success the moment the interface exists — it tells you
nothing about whether your traffic is really leaving through the tunnel. Four
ways you get burned:

1. **No route** — the tunnel is up but traffic still egresses on your real
   interface; your public IP never changed.
2. **DNS leak** — traffic tunnels fine, but your resolver still queries over
   your ISP, so *what* you look up leaks even when *where you connect from*
   doesn't.
3. **IPv6 leak** — the VPN is IPv4-only and your box happily reaches the world
   over native IPv6, straight past the tunnel.
4. **Tunnel drop mid-task** — the VPN flaps and everything silently reverts to
   your real IP while your work keeps running.

`wireguardian` refuses the green light unless (1)–(3) check out, and
`--killswitch` handles (4) by failing closed.

## How it works

- **Exit-IP proof** — records your public IP before and after `wg-quick up` and
  aborts if it didn't change.
- **DNS-leak canary** — resolves `whoami.akamai.net`, which replies with the
  egress IP of *your resolver*. If that's still your pre-VPN path, DNS is
  leaking even though your traffic isn't.
- **IPv6 probe** — a `curl -6` that flags any public IPv6 reachable outside the
  tunnel.
- **Kill-switch** — an `iptables` rule that drops every egress packet that is
  neither on the `wg` interface, nor the tunnel's own encrypted packets (matched
  by WireGuard's `fwmark`), nor destined to your local LAN. SSH and shared
  folders keep working; everything else is fail-closed. The applied `fwmark` is
  stashed under `/run` so the rule tears down cleanly even after the interface
  is gone.

## Requirements

```bash
sudo apt install -y wireguard iptables curl dnsutils
```

Put your WireGuard config at `/etc/wireguard/proton.conf` and `chmod 600` it —
it holds a private key. (Default tunnel name is `proton`; pass another as the
first argument.)

## Usage

```bash
git clone https://github.com/n4uu/wireguardian.git
cd wireguardian
./wireguardian up --killswitch     # bring up, prove no leaks, fail closed
./wireguardian status              # UP? kill-switch on? exit IP + country
./wireguardian check               # re-run the leak checks any time
./wireguardian down                # remove kill-switch and drop the tunnel
```

## Security notes

- **Your `.conf` is a credential.** It contains a private key — `chmod 600`,
  never commit it (`.gitignore` here blocks `*.conf`/`*.key`), and don't leave
  it in a shared or cloud-synced folder.
- **Know the AUP.** Most VPN providers prohibit port scanning over their
  network, and exit IPs are shared and often blocklisted. Only test targets you
  own or are authorised to.
- With the kill-switch on, a tunnel drop **cuts your internet on purpose** —
  that's the whole point. `./wireguardian down` restores it.

## License

MIT © n4uu
