# fatedier/frp

A fast Go reverse proxy that exposes local services behind NAT/firewalls to the internet — the canonical "ngrok alternative" in OSS.

## What it is

A Go-based reverse proxy that consists of a server (frps, run on a public host) and a client (frpc, run on the machine you want to expose). The client opens an outbound connection to the server; incoming traffic on the public endpoint is forwarded down that tunnel to the local service. Supports TCP, UDP, HTTP, HTTPS, STCP (secret TCP), P2P modes. Apache 2.0 licensed.

## Key features

- TCP / UDP / HTTP / HTTPS / STCP / XTCP tunnel types.
- Custom domain support for HTTP(S) tunnels with TLS termination at the frps server.
- Authentication via tokens, OIDC, mTLS.
- Bandwidth limits, load balancing across multiple frpcs.
- Web admin dashboard for tunnel monitoring.
- Cross-platform binaries (Linux, macOS, Windows, ARM).
- Apache 2.0 licensed.

## Tech stack

- Go primary.
- Single binary per side (frps server + frpc client).

## When to reach for it

- You're exposing a self-hosted service from home to the internet and you control a public VPS.
- You're an SRE / homelab user who wants the OSS alternative to ngrok / Cloudflare Tunnel.
- You need TCP / UDP / non-HTTP protocols, which most managed alternatives don't support.

## When *not* to reach for it

- You want zero-setup managed — ngrok, Cloudflare Tunnel are easier.
- You don't have a public-IP machine to run frps on.

## Maturity signal

107k stars, 15k forks, Apache 2.0, actively maintained. 9+ years. Open-issues count of 45 is exceptionally low.

## Alternatives

- ngrok — commercial, managed, faster setup.
- Cloudflare Tunnel — free, managed, requires Cloudflare-managed DNS.
- tailscale Funnel — peer-to-peer-flavored alternative.

## Tags

golang, reverse-proxy, networking, tunnel, nat, self-hosted, apache-license, p2p
