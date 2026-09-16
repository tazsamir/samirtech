---
title: "Local HTTPS Without Exposing My Homelab to the Internet"
date: 2026-09-16T03:00:00+01:00
draft: true
description: "How Caddy, CoreDNS and DNS-01 certificates give private services friendly names and trusted HTTPS without opening them to the internet."
tags: [homelab, networking, caddy, dns, self-hosting]
---

Remembering an address and a different port for every service gets old quickly. I wanted names I could bookmark, HTTPS that worked without certificate warnings, and no accidental public access to private applications.

Those are three separate problems. DNS supplies the address, TLS verifies and encrypts the connection, and routing and firewall rules determine who can reach the service. Solving one does not automatically solve the others.

## The layout I use

My always-on Docker host runs Caddy as the reverse proxy and CoreDNS for private service-name resolution. The applications keep their existing direct ports while the proxy provides the friendlier route.

```text
Phone or computer
  ├─ DNS query → local resolver → private proxy address
  └─ HTTPS request → Caddy → application on its existing port

Certificate renewal
  └─ Caddy → DNS-provider API → temporary public DNS challenge
```

For example, `photos.example.com` could resolve to the proxy's private address on the home network. The browser connects to Caddy, which forwards the request to the photo application. The example names here are placeholders, not my private configuration.

## A public certificate does not require a public application

ACME DNS-01 validation proves control of a domain using a DNS TXT record. The certificate authority checks that record; it does not need to reach the private photo application.

Caddy needs a build with the appropriate DNS-provider module. In my setup that is the Cloudflare module, with a restricted API token supplied through a private environment file. The token must not appear in Git, examples, screenshots or command output.

This lets the proxy obtain a publicly trusted wildcard certificate without opening an inbound web port solely for certificate validation. It still needs outbound access to the certificate authority and DNS provider. Working renewal also depends on the token remaining valid and the proxy's state surviving container replacement.

The certificate does not enforce privacy. Router port forwards, firewall rules, tunnels and application authentication must still be checked separately. Some services may have deliberately configured remote access; adding a local certificate should not create or change that access accidentally.

Publicly trusted certificates are recorded in certificate-transparency logs. A wildcard can avoid listing each individual service hostname, but the certificate's domain name is not secret.

## Why the client DNS setting matters

A correct CoreDNS record is useful only if the device actually asks a resolver that knows about it.

My workstation routes the relevant domain to the local resolver. Other clients need an equivalent arrangement, normally through the network's DNS policy. A phone using a different resolver, an encrypted-DNS setting or a VPN may follow another path.

That explains a common failure: the service works by its direct address, but its friendly name does not. Before changing the application, check which resolver the client is using.

Do not rely on a public secondary DNS server as a fallback for private records. Clients do not necessarily consult servers in the order expected, and an authoritative negative answer is not a request to try somewhere else.

When extending private resolution for a domain that also hosts a public website or email, preserve those public records. Only the intended service names should receive private answers; other queries need the correct forwarding path.

## Why not just use a private certificate authority?

That is a valid alternative. Names under `home.arpa` can use a private CA, but each client must trust that CA. Trust stores may differ between browsers and operating systems, and some devices make importing a CA inconvenient.

For my public-domain service names, DNS-01 certificates avoid that extra trust installation. Clients still need the right DNS and network access. My older private-CA aliases are a separate naming and trust arrangement, not interchangeable certificates.

Avoid inventing `.local` names for ordinary unicast DNS: `.local` is reserved for multicast DNS.

## Keep the original access route during the change

I kept direct application ports available while introducing the proxy. That gave me a way to distinguish a proxy problem from an application problem without rebuilding the application.

This is a migration safeguard, not a reason to expose every port widely. Direct access and proxied access each need an intentional firewall and authentication policy.

The proxy should route only explicitly configured hostnames. Adding a wildcard certificate must not silently turn every management interface into a reachable website.

## Checks that matter

For a setup like this, I would verify each layer separately:

1. **DNS:** the client's normal resolver returns the intended private address.
2. **TLS:** the browser or command-line client accepts the certificate without bypassing verification.
3. **Routing:** Caddy sends the request to the correct application.
4. **Application behavior:** login, redirects, uploads and any application-specific connections still work.
5. **Continuity:** existing direct routes and unrelated services still behave as before.
6. **Exposure:** router, firewall and tunnel configuration match the intended access policy.

These example commands are checks, not installation instructions:

```bash
# Replace the placeholder hostname with your own service.
getent ahosts photos.example.com
curl --head --show-error https://photos.example.com/
```

Do not add `--insecure` to make a certificate error disappear. A redirect or authentication response may be expected; a successful HTTP response alone does not test every application feature. Some applications do not support HEAD requests, so use a normal browser request when appropriate.

This article describes the deployed design, not a fresh security audit of every service or a claim that every household device has been configured.

## The trade-off

A shared proxy and resolver make access simpler, but they become shared dependencies. If either fails, several friendly URLs can stop working even while the applications remain healthy.

Their configuration, certificate state and protected credentials belong in the recovery plan. Keep a private record of the direct access routes as well. The useful result is fewer addresses to remember—not a system that can only be repaired through the very proxy that has failed.

### Official references

- [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Caddy Cloudflare DNS module](https://github.com/caddy-dns/cloudflare)
- [Let's Encrypt challenge types](https://letsencrypt.org/docs/challenge-types/)
- [CoreDNS hosts plugin](https://coredns.io/plugins/hosts/)

My main lesson is simple: a trusted certificate proves a name, not a network boundary. Keep DNS, certificate trust and access control separate, then test them together from the device that will actually use the service.
