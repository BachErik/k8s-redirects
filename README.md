# k8s-redirects

> [!IMPORTANT]  
> This helm chart was mostly programmed by ai.

A minimal Helm chart that creates **HTTP 301/302 redirects** using **Traefik**.  
For each redirect you define, the chart creates:

- a Traefik **Middleware (RedirectRegex)**,
- an **Ingress** that attaches that middleware,
- a tiny **Service**/**Deployment** (e.g., `traefik/whoami`) to satisfy the Ingress backend (the redirect happens before the backend is hit).

> Works great with **cert-manager** (for TLS) and **external-dns** (for DNS automation).

## Requirements

- Kubernetes with Traefik v2 as Ingress controller and CRDs installed (you need Traefik’s `Middleware` CRD).
- Optional: cert-manager (if you want automatic TLS certificates).
- Optional: external-dns (if you want DNS records created from your Ingress).

## Quick start (Helm CLI)

```bash
helm repo add bacherik https://bacherik.github.io/k8s-redirects/
helm repo update

# values.yaml example below
helm install my-redirect bacherik/k8s-redirects -n redirect --create-namespace \
  -f values.yaml
````

### Example `values.yaml`

```yaml
redirects:
  - name: bluesky
    host: bluesky.lyre.works
    # This regex runs in Traefik's RedirectRegex middleware (full request URL).
    regex: "^https?://bluesky\\.lyre\\.works(?:/(.*))?$"
    # Use ${1} from the capture group above to carry the remainder of the path.
    replacement: "https://bsky.app/profile/lyre.works/${1}"
    permanent: true               # true=301, false=302
    image: traefik/whoami         # tiny backend; redirect happens before it responds
    port: 80
    # Optional TLS settings:
    # tlsSecretName: ""           # override generated "<host>-tls"
    issuer: letsencrypt           # if using cert-manager (ClusterIssuer name)

# Global defaults
ingressClassName: traefik
entryPoints: [web, websecure]
```

## How it works & limitations

* The redirect is done by **Traefik’s RedirectRegex middleware** (regex+replacement on the requested URL).
  You still need an Ingress (and a trivial backend Service/Deployment) so Traefik can attach the middleware.

* **One host per `redirect` item**: the Kubernetes Ingress `rules.host` is a single hostname.
  You can define **multiple redirects** if you need multiple hosts.

* **Wildcards**: many Ingress controllers (including NGINX) support wildcard hosts like `*.example.com`. Traefik’s Kubernetes Ingress provider follows the Ingress spec; wildcard support is controller-specific. If you use wildcards, ensure your **DNS** also has a wildcard record and your **external-dns** provider supports it.
  See: Kubernetes/NGINX wildcard host docs and Traefik docs on routing/middlewares.

* **Why a backend at all?** Traefik must attach the middleware to a router; the router points to a service. The redirect happens before the backend responds, so a tiny image (like `traefik/whoami`) is fine.

## Values reference

| Key                         | Type           | Required | Default            | Description                                                                                        |
| --------------------------- | -------------- | -------- | ------------------ | -------------------------------------------------------------------------------------------------- |
| `redirects[].name`          | string         | yes      | —                  | Logical name for your redirect (used only in comments/examples).                                   |
| `redirects[].host`          | string (DNS)   | yes      | —                  | The Ingress host that should be redirected.                                                        |
| `redirects[].regex`         | string (regex) | yes      | —                  | Regex applied by Traefik’s RedirectRegex middleware against the requested URL. Use capture groups. |
| `redirects[].replacement`   | string         | yes      | —                  | Replacement with `${n}` groups. Usually your target URL + `${1}` to carry the path.                |
| `redirects[].permanent`     | bool           | yes      | —                  | `true` → 301, `false` → 302.                                                                       |
| `redirects[].image`         | string         | yes      | —                  | Lightweight container image for the backend (never actually used if redirect fires).               |
| `redirects[].port`          | int            | yes      | —                  | Container/service port.                                                                            |
| `redirects[].tlsSecretName` | string         | no       | `<host>-tls`       | TLS secret to use; leave empty to use generated default.                                           |
| `redirects[].issuer`        | string         | no       | —                  | cert-manager ClusterIssuer/Issuer name; if set, the annotation is added.                           |
| `ingressClassName`          | string         | no       | `traefik`          | Ingress class name to use.                                                                         |
| `entryPoints`               | list           | no       | `[web, websecure]` | Traefik entrypoints for the router annotation.                                                     |

## FAQ

* **Can I redirect multiple subdomains with one item?**
  Not with standard Kubernetes Ingress: you must list each host (or use a wildcard host if your controller + DNS support it). The regex in the middleware is for URL rewriting/redirecting, **not** for selecting which host gets routed.

* **Do I need cert-manager?**
  No, but TLS is easy with it. If `issuer` is empty, the chart won’t add the cert-manager annotation. You can also provide a pre-created secret via `tlsSecretName`.

## License

GPLv3. See [`LICENSE`](/LICENSE).