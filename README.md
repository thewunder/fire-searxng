# fire-searxng

Run a private [SearXNG](https://github.com/searxng/searxng) metasearch
instance and a self-hosted [Firecrawl](https://github.com/firecrawl/firecrawl)
extract API, orchestrated with [Lando](https://lando.dev/).

## What is included?

| Name | Description | Image |
|------|-------------|-------|
| [SearXNG](https://github.com/searxng/searxng) | Private metasearch engine | [`docker.io/searxng/searxng:latest`](https://hub.docker.com/r/searxng/searxng) |
| [Valkey](https://github.com/valkey-io/valkey) | In-memory store for SearXNG | [`docker.io/valkey/valkey:8-alpine`](https://hub.docker.com/r/valkey/valkey) |
| [Firecrawl](https://github.com/firecrawl/firecrawl) | Self-hosted scrape/extract API (`:3002`, `https://firecrawl.lndo.site`) | [`ghcr.io/firecrawl/firecrawl:latest`](https://github.com/firecrawl/firecrawl/pkgs/container/firecrawl) + Playwright, Redis, RabbitMQ, `nuq-postgres` |

Firecrawl is what Hermes uses for `web_extract`. Keep SearXNG for search. Point Hermes at the published port:

```yaml
# ~/.hermes/config.yaml
web:
  backend: searxng
  extract_backend: firecrawl
```

```bash
# ~/.hermes/.env
FIRECRAWL_API_URL=http://localhost:3002
```

Rebuild only the Firecrawl services after changing them:

```sh
lando rebuild -s firecrawl -s firecrawl-playwright -s firecrawl-redis -s firecrawl-rabbitmq -s firecrawl-postgres -y
```

## How to use it

1. [Install Lando](https://lando.dev/download/) and Docker.
2. Get fire-searxng:
   ```sh
   cd /usr/local
   git clone https://github.com/thewunder/fire-searxng.git
   cd fire-searxng
   ```
3. Generate the SearXNG secret key:
   ```sh
   sed -i "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml
   ```
   On a Mac: `sed -i '' "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml`
4. Edit [`searxng/settings.yml`](https://github.com/searxng/searxng-docker/blob/master/searxng/settings.yml) to taste.
5. Start the stack:
   ```sh
   lando start
   ```
   SearXNG is at `https://search.lndo.site`. Firecrawl is at
   `https://firecrawl.lndo.site` and `http://localhost:3002`.

> [!NOTE]
> Windows users can generate the SearXNG secret key with:
> ```powershell
> $randomBytes = New-Object byte[] 32
> (New-Object Security.Cryptography.RNGCryptoServiceProvider).GetBytes($randomBytes)
> $secretKey = -join ($randomBytes | ForEach-Object { "{0:x2}" -f $_ })
> (Get-Content searxng/settings.yml) -replace 'ultrasecretkey', $secretKey | Set-Content searxng/settings.yml
> ```

## Configuration

SearXNG reads [`searxng/settings.yml`](searxng/settings.yml), mounted at
`/etc/searxng`. The public base URL is `https://search.lndo.site/`
(`SEARXNG_BASE_URL` in [`.lando.base.yml`](.lando.base.yml)).

Firecrawl's environment, published ports, and service dependencies live in
[`.lando.base.yml`](.lando.base.yml). It uses its own Redis, RabbitMQ, and
Postgres so SearXNG's Valkey is left alone. `SEARXNG_ENDPOINT` is
`http://searxng:8080`.

## Start with systemd

You can skip this if you don't use systemd. The template is a **user** unit
(runs as your account, not root) so Lando can reach your Docker socket.

1. Copy the service template into your user systemd directory:
   ```sh
   mkdir -p ~/.config/systemd/user
   cp fire-searxng.service.template ~/.config/systemd/user/fire-searxng.service
   ```
2. Edit `WorkingDirectory` in that file if the repo is not at
   `~/IdeaProjects/fire-searxng`.
3. Enable lingering (starts the unit at boot without a graphical login) and
   enable the unit:
   ```sh
   loginctl enable-linger "$USER"
   systemctl --user daemon-reload
   systemctl --user enable --now fire-searxng.service
   ```

Check status with `systemctl --user status fire-searxng.service`. Logs:
`journalctl --user -u fire-searxng.service -f`.

## Update

Pull current images for SearXNG, Valkey, and Firecrawl:

```sh
lando rebuild
```
