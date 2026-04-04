# ollama-searxng

Create a new OpenWebUI, that runs local LLMs via Ollama, and secure local search using SearXNG instance in five minutes using Lando

## What is included?

| Name                                          | Description | Docker image                                                                | Dockerfile                                                                                    |
|-----------------------------------------------| -- |-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| [Openwebu-UI](https://openwebui.com/)         | UI for chatting with LLMs | [ghcr.io/open-webui/open-webui:main](https://github.com/open-webui/open-webui/pkgs/container/open-webui)  | [Dockerfile](https://github.com/open-webui/open-webui/blob/main/Dockerfile)                                                                                |
| [SearXNG](https://github.com/searxng/searxng) | SearXNG by itself                                              | [docker.io/searxng/searxng:latest](https://hub.docker.com/r/searxng/searxng) | [Dockerfile](https://github.com/searxng/searxng/blob/master/Dockerfile)                       |
| [Ollama](https://ollama.com/) | For running local LLMs | [docker.io/ollama/ollama](https://hub.docker.com/r/ollama/ollama) | [Dockerfile](https://github.com/ollama/ollama/blob/main/Dockerfile)                           |
| [Valkey](https://github.com/valkey-io/valkey) | In-memory database                                             | [docker.io/valkey/valkey:8-alpine](https://hub.docker.com/r/valkey/valkey)  | [Dockerfile](https://github.com/valkey-io/valkey-container/blob/mainline/Dockerfile.template) |

## How to use it

1. [Install lando](https://lando.dev/download/)
2. Get ollama-searxng
  ```sh
  cd /usr/local
  git clone https://github.com/thewunder/ollama-searxng.git
  cd ollama-searxng
  ```
3. Edit the [.env](https://github.com/searxng/searxng-docker/blob/master/.env) file to set the hostname and an email
4. Generate the secret key `sed -i "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml`  
   On a Mac: `sed -i '' "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml`
5. Edit [searxng/settings.yml](https://github.com/searxng/searxng-docker/blob/master/searxng/settings.yml) according to your needs

> [!NOTE]
> Windows users can use the following powershell script to generate the secret key:
> ```powershell
> $randomBytes = New-Object byte[] 32
> (New-Object Security.Cryptography.RNGCryptoServiceProvider).GetBytes($randomBytes)
> $secretKey = -join ($randomBytes | ForEach-Object { "{0:x2}" -f $_ })
> (Get-Content searxng/settings.yml) -replace 'ultrasecretkey', $secretKey | Set-Content searxng/settings.yml
> ```

### Start Ollama + SearXNG with systemd

You can skip this step if you don't use systemd.
1. Copy the service template file:
   ```sh
   cp ollama-searxng.service.template ollama-searxng.service
   ```
  
2. Edit the content of ```youruser``` in the ```ollama-searxng.service``` and the correct path if different than ```/usr/local/ollama-searxng```)
   
3. Enable the service:
   ```sh
   systemctl enable $(pwd)/ollama-searxng.service
   ```

4. Start the service:
   ```sh
   systemctl start ollama-searxng.service
   ```

**Note:** Ensure the service file path matches your installation directory before enabling it.

## Multi Architecture Docker images

Supported architecture:

- amd64
- arm64
- arm/v7

## Update

Just run a lando rebuild to update ollama, searxng and openwebui to the latest versions.

```sh
lando rebuild
```
