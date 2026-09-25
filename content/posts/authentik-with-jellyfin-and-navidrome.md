---
title: "Authentik with Jellyfin and Navidrome: Two Different Paths to One Identity"
date: 2026-09-25
draft: false
description: "What I learned while connecting Jellyfin and Navidrome to Authentik without breaking native media clients."
tags:
  - authentik
  - jellyfin
  - navidrome
  - caddy
  - sso
  - homelab
---

Jellyfin and Navidrome can use the same Authentik directory, but they do not authenticate in the same way.

In my setup, Jellyfin uses LDAP. Navidrome's web interface sits behind Authentik and Caddy, with the authenticated username passed in an HTTP header. Navidrome's OpenSubsonic API remains separate so applications such as Symfonium and Feishin can use native credentials.

That distinction is the key to making the arrangement understandable and safe.

## The three authentication paths

```text
Jellyfin -> LDAP plugin -> Authentik LDAP outpost

Browser -> Caddy -> Authentik forward auth -> Navidrome

OpenSubsonic client -> /rest/* -> Navidrome credentials
```

Trying to force all three through one browser-oriented SSO flow causes problems. Native media clients expect an API response, not an HTML login page or a redirect.

## Jellyfin: LDAP needs an LDAP service

Installing Jellyfin's LDAP plugin is only the client half of the job. Authentik must already have:

- an LDAP provider;
- an LDAP outpost serving that provider;
- a bind account for Jellyfin;
- a user search base and attribute that match the directory; and
- an explicit rule for which users may log in.

The outpost must be reachable from Jellyfin's own network namespace. A connection test from a laptop does not prove that a container can resolve or reach the same endpoint.

The useful troubleshooting order is:

1. confirm that the LDAP outpost is healthy;
2. test DNS and the LDAP port from Jellyfin's network;
3. verify TLS trust, if TLS is enabled;
4. verify the bind DN and secret;
5. check the user base, username attribute and filter; and
6. inspect both the Jellyfin and outpost logs during one login attempt.

If every user fails, the transport, bind or search configuration is probably wrong. If valid users can authenticate but the wrong users are also admitted, the connection works and the scope is too broad.

## Use an allow-list, not a guest deny-list

I did not want every Authentik account to inherit Jellyfin access. In particular, guest identities needed to remain restricted.

The safer model is an approved media group. The Authentik provider can expose only the intended users, while Jellyfin's LDAP configuration also requires membership of that group. This is easier to audit than maintaining a growing list of exclusions.

A generic user filter might resemble:

```ldap
(&(objectClass=<USER_OBJECT_CLASS>)(<USERNAME_ATTRIBUTE>={0}))
```

A group restriction depends on the schema exposed by the installed Authentik version and the syntax expected by the Jellyfin plugin. Copying a filter from another directory without inspecting the actual attributes is unreliable.

Test four cases before considering it finished:

- an approved user with the correct password;
- a valid Authentik user outside the media group;
- a guest account; and
- an approved user with the wrong password.

Only the first should succeed.

LDAP authentication also does not define every Jellyfin policy. Library access and administrator privileges still need to be reviewed on the resulting Jellyfin user.

## Navidrome: forward authentication through Caddy

Navidrome can trust a username header supplied by an approved reverse proxy. In outline, its external-authentication settings look like this:

```yaml
environment:
  ND_EXTAUTH_USERHEADER: "X-Authentik-Username"
  ND_EXTAUTH_TRUSTEDSOURCES: "<CADDY_SOURCE_ADDRESS>/32"
  ND_EXTAUTH_AUTOUSERCREATION: "true"
```

Check the names against the installed Navidrome release before applying them.

Caddy asks Authentik to authorise the browser request and copies the returned identity headers:

```caddyfile
music.example.invalid {
    route {
        reverse_proxy /outpost.goauthentik.io/* <AUTHENTIK_OUTPOST>

        @browser not path /rest/* /share/* /outpost.goauthentik.io/*
        forward_auth @browser <AUTHENTIK_OUTPOST> {
            uri /outpost.goauthentik.io/auth/caddy
            copy_headers X-Authentik-Username X-Authentik-Groups X-Authentik-Email
        }

        reverse_proxy <NAVIDROME_UPSTREAM>
    }
}
```

This is an architectural example, not a drop-in configuration. The recommended outpost route can change between Authentik versions.

With automatic user creation enabled, the first successful browser login creates a corresponding Navidrome account. New accounts should still be checked to ensure they have not received administrator rights accidentally.

## The trusted-source trap

The username header is not proof of identity by itself. Any client can try to send `X-Authentik-Username`.

It becomes trustworthy only when:

1. Caddy ignores or replaces any client-supplied identity header;
2. Authentik validates the request;
3. Caddy inserts the identity returned by Authentik; and
4. Navidrome accepts the header only from Caddy's actual network source.

The last point caused the most misleading failure. The address Navidrome sees may be a container address, bridge gateway or host address depending on the network path. It is not necessarily the address used in a browser.

When the trusted source is wrong, Authentik succeeds and Caddy forwards the header, but Navidrome ignores it and presents its own login screen. Navidrome's log entry about an untrusted source is more useful than guessing.

Trust the narrowest stable address or CIDR possible. Do not use `0.0.0.0/0` simply to make the warning disappear, and do not expose Navidrome directly to an untrusted network while it accepts proxy identity headers.

## Keep `/rest` outside browser SSO

OpenSubsonic clients use Navidrome's `/rest` API. They do not complete Authentik's browser redirect flow.

Caddy therefore needs a narrow exception for the API path before applying forward authentication to browser routes. This does not make the API anonymous: Navidrome still validates the OpenSubsonic username and password.

If `/rest` is sent through forward authentication, clients may receive an Authentik page or redirect instead of the expected API response.

## One user, two passwords

An account created by the SSO flow can use Authentik in a browser without having a usable Navidrome password. Native clients need a password known to Navidrome.

The setup for a new user is therefore:

1. sign in through Authentik in a browser, creating the Navidrome account;
2. set a separate password for that account in Navidrome; and
3. use the Navidrome username and password in the OpenSubsonic client.

For Symfonium, select its Subsonic or OpenSubsonic provider. In Feishin, OpenSubsonic mode worked with this arrangement; its Navidrome-native mode followed a different path and conflicted with the proxy SSO configuration.

The Authentik password should not be entered into those clients. SSO and API credentials serve different protocols and should remain separate.

## Verification checklist

### Jellyfin

- The LDAP outpost is reachable from Jellyfin.
- An approved user can sign in.
- Valid but unauthorised and guest users cannot sign in.
- A wrong password fails.
- Local Jellyfin library and administrator policies remain correct.

### Navidrome browser

- A signed-out browser is redirected to Authentik.
- A successful login reaches the expected Navidrome account.
- A forged client identity header cannot bypass Authentik.
- Navidrome trusts only the source from which Caddy actually connects.
- Direct access is blocked or uses Navidrome's own authentication.

### OpenSubsonic

- `/rest/*` does not redirect to Authentik.
- The client works with a Navidrome password.
- An incorrect Navidrome password fails.
- The SSO exemption does not include unrelated browser routes.

## The lesson

A shared identity system does not require every application to use the same protocol.

Jellyfin needs a correctly scoped LDAP provider and outpost. Navidrome's browser interface needs a trusted reverse proxy and carefully controlled identity header. OpenSubsonic clients need Navidrome's native API credentials.

Once those paths are treated separately, the setup stops feeling like one mysterious login problem and becomes three small, testable boundaries.
