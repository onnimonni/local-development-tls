# trusted-https-certificate-generator-action

A GitHub Action that issues a publicly trusted Let's Encrypt certificate and
keeps it as an artifact of your **private** repository: no self-signed certificates,
no local CA to install. Use it for development hostnames that point at `127.0.0.1`
(`*.dev.example.com`), internal services, test environments or anything else that
wants HTTPS.

- Validation is DNS-01 through Cloudflare, with [lego](https://go-acme.github.io/lego/)'s
  Docker image pinned by digest, so the names don't need to be reachable from the
  internet.
- Every run issues a new certificate with a new key and uploads it as one PEM file,
  the chain followed by the key: the artifact `https-certificate.pem`, unzipped,
  kept 90 days.
- It refuses to run in a public repository: public repositories' artifacts and logs
  are public.

## Setup

1. **DNS** in Cloudflare for the names, e.g. `*.dev.example.com` A `127.0.0.1` (DNS
   only, not proxied) for local development. Some routers' DNS rebinding protection
   drops answers pointing at 127.0.0.1: allow the domain there.
2. **Cloudflare API token** (My Profile → API Tokens → Create Token → Custom token):
   **Zone → DNS → Edit** (the `_acme-challenge` TXT records) and **Zone → Zone → Read**
   (finding the zone); Zone Resources: Include → Specific zone → your zone. It can't
   be limited to TXT records: give the names a zone of their own to keep it from your
   other records.
3. **In the repository**, from any directory:

   ```sh
   REPO=owner/repo
   gh variable set HTTPS_CERTIFICATE_DOMAINS -R "$REPO" --body 'dev.example.com,*.dev.example.com'
   gh secret set HTTPS_CERTIFICATE_CLOUDFLARE_TOKEN -R "$REPO"   # paste the token
   gh api -X PUT "repos/$REPO/contents/.github/workflows/https-certificate.yml" \
     -f message="Add HTTPS certificate workflow" \
     -f content="$(gh api -H 'Accept: application/vnd.github.raw' repos/onnimonni/trusted-https-certificate-generator-action/contents/example.yml | base64 | tr -d '\n')"
   gh workflow run https-certificate.yml -R "$REPO"
   ```

   This commits [`example.yml`](example.yml) to the default branch (your `gh`
   login needs the `workflow` scope: `gh auth refresh -s workflow`) and runs it
   once:

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
         - uses: onnimonni/trusted-https-certificate-generator-action@8544f7e1617ddf32a48b81884bbcb7a3d2ed5379 # v1.0.0
           with:
             domains: ${{ vars.HTTPS_CERTIFICATE_DOMAINS }}
             cloudflare-token: ${{ secrets.HTTPS_CERTIFICATE_CLOUDFLARE_TOKEN }}
   ```

   Monthly runs keep a 90-day certificate with 60 days to spare; run it by hand
   (`gh workflow run https-certificate.yml`) after changing the names. A wildcard
   covers one label. It needs a Linux runner with Docker (`ubuntu-latest` has it).

## Inputs

| Input | Default | |
|---|---|---|
| `domains` | | Comma-separated names to certify. |
| `cloudflare-token` | | The Cloudflare API token; pass it from a secret. |
| `server` | `letsencrypt` | ACME server: a URL or a lego shortcode. Try `letsencrypt-staging` first: untrusted certificates, no rate limits (production allows 50 certificates per domain a week). |
| `retention-days` | `90` | How long the artifact is kept. |

## Using the certificate

Anyone who can read the repository can download it with `gh`:

```sh
id=$(gh api "repos/OWNER/REPO/actions/artifacts?name=https-certificate.pem" \
  --jq '[.artifacts[] | select(.expired | not)] | sort_by(.created_at) | last | .id')
gh api "repos/OWNER/REPO/actions/artifacts/$id/zip" > https-certificate.pem
```

The endpoint says `zip`, but the file comes back as is. `gh run download` doesn't
work: it expects zip files.

## License

MIT
