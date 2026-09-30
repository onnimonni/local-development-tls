# trusted-https-certificate-to-artifacts-action

> [!CAUTION]
> ❗ **Generates real trusted https certificates with private keys into Github repo
> artifacts. The action refuses to run in a public repository, whose artifacts and
> logs anyone can download. Everyone who can read your private repository can read
> the key.**

A GitHub Action that issues a publicly trusted Let's Encrypt certificate and
keeps it as an artifact of your **private** repository: no self-signed certificates,
no local CA to install. Use it for development hostnames that point at `127.0.0.1`
(`*.example-dev.com`), internal services, test environments or anything else that
wants HTTPS.

- Validation is DNS-01 through Cloudflare, with [lego](https://go-acme.github.io/lego/)'s
  Docker image pinned by digest, so the names don't need to be reachable from the
  internet.
- Every run issues new certificates with new keys: one per registered domain (and
  per 100 names), as combined PEM files (the chain followed by the key), zipped into
  the artifact `https-certificate`, kept 90 days. The run's summary lists them:

  | Certificate | Names |
  |---|---|
  | `different-domain.com.pem` | `*.different-domain.com` |
  | `example-dev.com.pem` | `{api,app,simulator}.example-dev.com` |

## Setup

1. **Buy a separate domain** just for this, e.g. `example-dev.com`, and add it to
   Cloudflare (or buy it from [Cloudflare Registrar](https://domains.cloudflare.com)).

> [!WARNING]
> Cloudflare can't limit the API token to TXT records: it can change every DNS record
> of its zone. With a domain of its own, a leaked token can't touch your real website
> or email.

2. **DNS record** for the names, e.g. `*.example-dev.com` A `127.0.0.1` (DNS only, not
   proxied) for local development.

> [!NOTE]
> Some routers' DNS rebinding protection (Fritzbox, pfSense, dnsmasq
> `stop-dns-rebind`) drops answers pointing at 127.0.0.1: allow the domain there.

3. **[Cloudflare API token](https://dash.cloudflare.com/profile/api-tokens)** (Create
   Token → Custom token) with **Zone → DNS → Edit** and **Zone → Zone → Read**, for
   that zone only.

4. **In the repository**:

   ```sh
   cd your-project-folder
   gh variable set HTTPS_CERTIFICATE_DOMAINS --body '{app,api,simulator}.example-dev.com,*.different-domain.com'
   gh secret set HTTPS_CERTIFICATE_CLOUDFLARE_TOKEN   # paste the token
   gh api -X PUT 'repos/{owner}/{repo}/contents/.github/workflows/https-certificate.yml' \
     -f message="Add HTTPS certificate workflow" \
     -f content="$(gh api -H 'Accept: application/vnd.github.raw' repos/onnimonni/trusted-https-certificate-to-artifacts-action/contents/example.yml | base64 | tr -d '\n')"
   gh workflow run https-certificate.yml
   ```

   This commits [`example.yml`](example.yml) to the default branch and runs it once:

   ```yaml
   name: HTTPS certificate

   on:
     schedule:
       - cron: "17 4 1 * *"
     workflow_dispatch:

   permissions: {}

   jobs:
     renew:
       runs-on: ubuntu-latest
       steps:
         - uses: onnimonni/trusted-https-certificate-to-artifacts-action@609a84868385b96923ca2d04cf2423729a92e222 # v2.0.0
           with:
             domains: ${{ vars.HTTPS_CERTIFICATE_DOMAINS }}
             cloudflare-token: ${{ secrets.HTTPS_CERTIFICATE_CLOUDFLARE_TOKEN }}
   ```

> [!TIP]
> Names are comma-separated and `{a,b}` expands like in bash: the example above
> certifies `app.example-dev.com`, `api.example-dev.com`, `simulator.example-dev.com`
> and `*.different-domain.com`. Both domains need to be in Cloudflare.

> [!IMPORTANT]
> Your `gh` login needs the `workflow` scope to commit a workflow:
> `gh auth refresh -s workflow`.

> [!TIP]
> Try it with `server: letsencrypt-staging` first: untrusted certificates, but no
> rate limits (production allows 50 certificates per domain a week).

Monthly runs keep a 90-day certificate with 60 days to spare; run it by hand
(`gh workflow run https-certificate.yml`) after changing the names. A wildcard
covers one label: `*.example-dev.com` doesn't cover `a.b.example-dev.com`. It needs
a Linux runner with Docker (`ubuntu-latest` has it).

## Inputs

| Input | Default | |
|---|---|---|
| `domains` | | Comma-separated names to certify; `{a,b}` expands like in bash. One certificate per registered domain and per 100 names (Let's Encrypt's limit), at most 20. |
| `cloudflare-token` | | The Cloudflare API token; pass it from a secret. |
| `server` | `letsencrypt` | ACME server: a URL or a lego shortcode such as `letsencrypt-staging`. |
| `retention-days` | `90` | How long the artifact is kept. |

## Using the certificates

Anyone who can read the repository can download them with `gh`:

```sh
cd your-project-folder
gh run download --name https-certificate --dir certs
```

`certs/` then holds one `.pem` per certificate, named after its domain, e.g.
`example-dev.com.pem` (`example-dev.com-2.pem` past 100 names). Servers take them
as they are:

| Server | |
|---|---|
| HAProxy | `crt certs/`: every file, chosen by hostname |
| Caddy | `load_folders` / `tls { load certs }`: every file, chosen by hostname |
| nginx | `ssl_certificate` and `ssl_certificate_key` both pointing at one file |
| Apache | `SSLCertificateFile` pointing at one file |

> [!NOTE]
> The files hold private keys: `chmod 600 certs/*.pem`.

## License

MIT
