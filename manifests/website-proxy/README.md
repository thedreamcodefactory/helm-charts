# dreamcodefactory.com Kubernetes Ingress Proxy

This manifest configures a Kubernetes-based reverse proxy that sits in front of two external hosting providers and serves all traffic under the `dreamcodefactory.com` domain.

## Overview

Instead of pointing DNS records directly at Webflow or GitHub Pages, all traffic is routed through an nginx ingress controller inside the cluster. This gives full control over TLS termination, URL rewrites, redirects, and host headers without changing anything on the upstream providers.

```
Browser
  └─► dreamcodefactory.com (nginx ingress)
        ├─► / and www.*         ──► dream-code-factory.webflow.io   (Webflow site)
        └─► /wiki/*             ──► thedreamcodefactory.github.io   (GitHub Pages docs)
```

---

## Resources

### ExternalName Services

Two headless Kubernetes Services act as DNS aliases that let the ingress controller address the external upstreams by an in-cluster name.

| Service | Points to |
|---|---|
| `webflow-upstream` | `dream-code-factory.webflow.io:443` |
| `github-pages-upstream` | `thedreamcodefactory.github.io:443` |

Because both upstreams only serve HTTPS, the services expose port 443.

---

### Ingress: `dreamcodefactory-webflow-proxy`

Handles all root traffic for both the apex domain and the `www` subdomain.

**TLS**
Cert-manager issues and renews a Let's Encrypt certificate (cluster issuer `letsencrypt-prod`) covering both `dreamcodefactory.com` and `www.dreamcodefactory.com`, stored in the secret `dreamcodefactory-tls`.

**www redirect**
Any request arriving on `www.dreamcodefactory.com` is permanently redirected (HTTP 301) to the equivalent path on the apex domain (`dreamcodefactory.com`). ACME challenge requests (`/.well-known/acme-challenge/`) are excluded from the redirect so certificate renewal continues to work.

**Upstream proxying**
Requests that are not redirected are forwarded over HTTPS to `webflow-upstream`. The `nginx.ingress.kubernetes.io/upstream-vhost` annotation overrides the `Host` header sent to Webflow to `dream-code-factory.webflow.io`, which is required because Webflow uses SNI-based virtual hosting. `proxy_ssl_server_name on` ensures the TLS handshake with Webflow also uses that hostname.

---

### Ingress: `dreamcodefactory-docs-proxy`

Handles traffic under the `/wiki/` path prefix on the apex domain.

**Path matching**
The path `/wiki(/|$)(.*)` captures everything under `/wiki/` using `ImplementationSpecific` matching (required for regex capture groups in nginx).

**URL rewriting**
Two rewrites are applied before the request is forwarded upstream:

1. `/wiki/...` is rewritten to `/docs/...` so that the public-facing URL can differ from the GitHub Pages layout.
2. A second rewrite normalises paths already starting with `/docs/` to prevent double-prefixing.

**Upstream proxying**
Traffic is forwarded over HTTPS to `github-pages-upstream`. The upstream vhost is set to `thedreamcodefactory.github.io` so GitHub Pages serves the correct site, and `proxy_ssl_server_name on` ensures SNI matches.

TLS termination reuses the same `dreamcodefactory-tls` secret as the main ingress.

---

## Key Annotations Reference

| Annotation | Effect |
|---|---|
| `cert-manager.io/cluster-issuer` | Triggers automatic TLS certificate issuance via Let's Encrypt |
| `nginx.ingress.kubernetes.io/backend-protocol: HTTPS` | Proxies to upstream over TLS (not plain HTTP) |
| `nginx.ingress.kubernetes.io/upstream-vhost` | Overrides the `Host` header sent to the upstream |
| `nginx.ingress.kubernetes.io/use-regex` | Enables regex path matching on the ingress rules |
| `nginx.ingress.kubernetes.io/server-snippet` | Injects raw nginx config (used for the www redirect logic) |
| `nginx.ingress.kubernetes.io/configuration-snippet` | Injects nginx config scoped to a location block (used for URL rewrites) |
| `ingress.kubernetes.io/preserve-host: "false"` | Allows the upstream-vhost annotation to override the Host header |

---

## Notes

- Both ingresses share a single TLS secret (`dreamcodefactory-tls`). Cert-manager keeps the certificate up to date automatically.
- The ACME challenge exclusion in the server snippet is essential. Without it, the www-to-apex redirect would intercept Let's Encrypt validation requests and break certificate renewal.
- `proxy_ssl_name` and `proxy_ssl_server_name on` appear in both ingresses. These configure the TLS SNI sent during the backend connection and must match the hostname the upstream expects, otherwise the upstream's TLS certificate validation will fail.
