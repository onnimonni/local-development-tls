# local-development-tls

A reusable GitHub Actions workflow that issues a publicly trusted Let's Encrypt
certificate for development hostnames that point at `127.0.0.1`, such as
`*.dev.example.com`, and keeps it as an artifact of your **private** repository.

Browsers, phone simulators, Node, curl and CI then trust your local HTTPS without
installing a local CA. The certificate and its key never enter git.

- Validation is DNS-01 through Cloudflare, with [lego](https://go-acme.github.io/lego/)'s
  Docker image pinned by digest.
- Every run issues a new certificate with a new key and uploads it as one PEM file,
  the chain followed by the key: the artifact `local-development-tls.pem`,
  unzipped, kept 90 days.
- It refuses to run in a public repository: public repositories' artifacts and logs
  are public.

## Setup

1. **DNS** in Cloudflare: `*.dev.example.com` A `127.0.0.1`, DNS only (not proxied).
   Some routers' DNS rebinding protection drops answers pointing at 127.0.0.1: allow
   the domain there.
2. **Cloudflare API token** (My Profile → API Tokens → Create Token → Custom token):
   **Zone → DNS → Edit** (the `_acme-challenge` TXT records) and **Zone → Zone → Read**
   (finding the zone); Zone Resources: Include → Specific zone → your zone. Save it as
   the repository secret `LOCAL_DEVELOPMENT_TLS_CLOUDFLARE_TOKEN`.
3. **Names** as the repository variable `LOCAL_DEVELOPMENT_TLS_DOMAINS`, comma-separated,
   e.g. `dev.example.com,*.dev.example.com`. A wildcard covers one label.
4. **The workflow**, e.g. `.github/workflows/local-development-tls.yml`:

   ```yaml
   name: Local development certificate

   on:
     schedule:
       - cron: "17 4 1 * *"
     workflow_dispatch:

   permissions: {}

   jobs:
     renew:
       uses: onnimonni/local-development-tls/.github/workflows/renew.yml@v1
       with:
         domains: ${{ vars.LOCAL_DEVELOPMENT_TLS_DOMAINS }}
       secrets:
         cloudflare-token: ${{ secrets.LOCAL_DEVELOPMENT_TLS_CLOUDFLARE_TOKEN }}
   ```

   Run it once by hand (Actions → Local development certificate → Run workflow).
   Monthly runs keep a 90-day certificate with 60 days to spare; run it by hand after
   changing the names.

## Inputs

| Input | Default | |
|---|---|---|
| `domains` | | Comma-separated names to certify. |
| `server` | `letsencrypt` | ACME server: a URL or a lego shortcode. Try `letsencrypt-staging` first: untrusted certificates, no rate limits (production allows 50 certificates per domain a week). |
| `retention-days` | `90` | How long the artifact is kept. |

| Secret | |
|---|---|
| `cloudflare-token` | The Cloudflare API token. |

## Using the certificate

Anyone who can read the repository can download it with `gh`:

```sh
id=$(gh api "repos/OWNER/REPO/actions/artifacts?name=local-development-tls.pem" \
  --jq '[.artifacts[] | select(.expired | not)] | sort_by(.created_at) | last | .id')
gh api "repos/OWNER/REPO/actions/artifacts/$id/zip" > local-development-tls.pem
```

The endpoint says `zip`, but the file comes back as is. `gh run download` doesn't
work: it expects zip files.

[lazy-cow-tree](https://github.com/onnimonni/devenv-lazy-cow-worktrees) reads it
this way and serves it for every worktree's hostnames.

## License

MIT
