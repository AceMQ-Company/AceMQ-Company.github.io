# Domains and GitHub Pages

How AceMQ's documentation domains are wired, what the options are for `acemq.net`,
and which one is recommended. Written down because the DNS work happens once, in
someone else's hands, and undoing it is more expensive than deciding carefully.

**Status: open.** `acemq.net` is registered but parked. Nothing below has been
applied yet.

## How it works today

| Domain | Serves | Configured by |
|---|---|---|
| `acemq.org` | This repo, the organisation site | `CNAME` file here |
| `acemq.org/<repo>/` | Every project's Pages site | inherited, nothing per-repo |
| `acemq.com` | Enterprise support | outside GitHub |
| `acemq.net` | nothing — parked at `192.64.119.113` | — |

`acemq.org` resolves to GitHub's four Pages addresses (`185.199.108–111.153`) and
has HTTPS enforced. Every project repo has no custom domain of its own, which is
why they all appear as paths underneath it.

## The constraint that decides everything

**Path nesting — `domain/<repo>/` — comes only from the organisation site, and an
organisation has exactly one.**

A project repo that sets a custom domain is served at the **root** of that domain.
It cannot host other repositories underneath it. So there is no way to reproduce
the `acemq.org/<repo>/` layout under `acemq.net` while the .NET repositories live
in `AceMQ-Company` — the second domain would have to belong to a second
organisation.

That single fact is what separates the options below. Everything else is detail.

## Options

### A — `acemq.net` points at the .NET library repository

`acemq.net` becomes the custom domain of `acemq-dotnet-amqp`, serving its
documentation at the apex. Any future .NET repository gets its own subdomain,
for example `examples.acemq.net`.

- One `CNAME` file, one set of DNS records.
- The visitor typing `acemq.net` lands on the .NET documentation immediately,
  with no landing page in the way. Today that is correct: the library *is* the
  .NET line.
- Growth costs one DNS record per repository, and gives subdomains rather than
  paths.

### B — a subdomain such as `docs.acemq.net`

Mechanically identical to A but as a `CNAME` record rather than apex `A` records,
which some DNS providers prefer. Leaves the apex free for a future landing page,
at the cost of the apex doing nothing in the meantime.

### C — a second organisation

A new organisation, say `AceMQ-DotNet`, whose organisation site takes `acemq.net`,
with the .NET repositories living inside it and serving at `acemq.net/<repo>/`.

This is the only option that reproduces the current layout, and it is the most
expensive one to operate. The duplication is not a file; it is an organisation:

- The repositories have to **move**. `github.com/AceMQ-Company/acemq-dotnet-amqp`
  becomes `github.com/AceMQ-DotNet/acemq-dotnet-amqp`. GitHub redirects the old
  URLs, but remotes, CI references, badges and the cross-links from the Maven and
  NuGet indexes all need updating by hand.
- Every organisation-level setting is configured twice: members and teams, base
  permissions, Actions policies, rulesets, Dependabot configuration, security
  policy, verified domains, and secrets — including the Slack delivery webhook.
- **Reusable workflows stop inheriting secrets.** `secrets: inherit` does not
  cross an organisation boundary. That has already cost us once, in a set of
  notification jobs that reported success while sending nothing, and every shared
  workflow would acquire the same failure mode permanently.
- Anyone browsing `AceMQ-Company` sees no .NET work at all.

Public repositories and their Actions minutes are free, so billing is not a
meaningful factor either way.

### D — a redirect to `acemq.org/acemq-dotnet-amqp/`

Cheapest of all, and the URL bar ends up showing `acemq.org`, which defeats the
point of having a separate .NET domain.

## Recommendation: A

1. There is exactly one .NET repository. A landing page in front of a single
   library is an extra click that informs nobody, and the library's own index
   page already introduces the .NET line and links back to `acemq.org`.
2. Subdomains absorb growth cleanly. A future examples or ASP.NET Core repository
   becomes `examples.acemq.net` — one record each, no migration of the apex, no
   broken links.
3. **A does not foreclose C.** If the .NET line grows to four or five
   repositories, or the branding needs to separate harder than it does now, the
   repositories can still be transferred and the domain moved. Starting with A
   costs one file and one DNS change; starting with C costs a permanent second
   organisation to find out whether it was needed.

C is the option that is easiest to *picture*, because it mirrors what `acemq.org`
already does. It is not the easiest to *run*. That distinction is the whole
argument.

The trigger for revisiting: **four or more .NET repositories**, or a decision to
present .NET as a separately branded product.

## What applying A requires

**DNS, by the domain administrator.** Apex records cannot be `CNAME` in standard
DNS, so either four `A` records or an `ALIAS`/`ANAME` if the registrar supports
one — the latter is preferable.

```
A     acemq.net       185.199.108.153
A     acemq.net       185.199.109.153
A     acemq.net       185.199.110.153
A     acemq.net       185.199.111.153
AAAA  acemq.net       2606:50c0:8000::153
AAAA  acemq.net       2606:50c0:8001::153
AAAA  acemq.net       2606:50c0:8002::153
AAAA  acemq.net       2606:50c0:8003::153
CNAME www.acemq.net   acemq-company.github.io.
```

The four IPv4 addresses are confirmed against what `acemq.org` resolves to today.
**Confirm the IPv6 set against GitHub's current documentation before applying it**
— it is quoted from memory, not from a live lookup.

**In the repository.** A `CNAME` file containing exactly `acemq.net` in
`acemq-dotnet-amqp`. It must match what DNS is configured for, or Pages refuses to
serve the site.

**Afterwards.** GitHub issues a Let's Encrypt certificate once DNS resolves —
minutes, occasionally up to a day — after which *Enforce HTTPS* becomes available
and should be switched on.

**Consequences to expect.** `acemq.org/acemq-dotnet-amqp/` stops serving once the
custom domain takes effect, so the .NET links on this site, in the Maven and NuGet
index READMEs, and in the .NET repository's own README all move to `acemq.net`.

## Domain verification is a separate matter, and overdue

The organisation has **no verified domains** (`is_verified: false`), for
`acemq.org` or anything else.

Without verification, a domain pointed at GitHub Pages can be claimed by another
GitHub account if the repository behind it is ever renamed, deleted, or has its
Pages site disabled — the subdomain-takeover class of problem. The result is
someone else's content served from our domain, under our certificate.

Verification is a `TXT` record (`_github-pages-challenge-AceMQ-Company`) added
under organisation settings, and it pins the domain to this organisation. It is
worth doing for `acemq.org` **and** `acemq.net` while the administrator is in the
DNS panel, independently of which option above is chosen.
