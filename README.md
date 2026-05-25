# Hostinger Hermes Agent + SuperGrok + xurl X API Setup

Beginner-friendly setup guide for running Hermes Agent on a Hostinger VPS, connecting it to SuperGrok/xAI OAuth, and adding `xurl` so the agent can call the X API.

This guide is based on a working Hostinger VPS setup where:

- Hermes Agent runs in Docker.
- Hermes is exposed over HTTPS through Traefik.
- SuperGrok/xAI OAuth works inside the Hermes container.
- `xurl` works inside the Hermes browser terminal.
- X API calls work after copying the local `xurl` OAuth credentials into the VPS container.

## What You Will Build

```text
Browser
  -> HTTPS Hermes URL
  -> Traefik
  -> Hermes Agent Docker container
  -> SuperGrok/xAI OAuth
  -> xurl
  -> X API
```

## Requirements

- Hostinger VPS with Docker and Docker Compose.
- Hostinger Hermes Agent container.
- A public hostname for Hermes, for example:

```text
hermes-agent.example.com
```

- A working Traefik container if you want HTTPS.
- SuperGrok/xAI account.
- X Developer account with Pay Per Use enabled.
- Some X API credits for real API calls.
- A local Windows machine for OAuth setup.

Do not commit or share:

- `.env`
- `.xurl`
- OAuth callback URLs
- API keys
- client secrets
- access tokens

## 1. Put Hermes Behind Traefik HTTPS

If you already have an n8n stack running Traefik on ports `80` and `443`, use that same Traefik instance for Hermes. Do not run a second Traefik container that also tries to own `80` and `443`.

Create one shared Docker network:

```bash
docker network create proxy
```

It is okay if Docker says the network already exists.

### n8n / Traefik Compose

Your Traefik service should be attached to the shared `proxy` network:

```yaml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy

volumes:
  traefik_data:
    external: true

networks:
  proxy:
    external: true
```

If n8n is in the same compose file, also attach n8n to `proxy` and add:

```yaml
labels:
  - traefik.docker.network=proxy
```

### Hermes Compose

Use this shape for Hermes:

```yaml
services:
  hermes-agent:
    image: ghcr.io/hostinger/hvps-hermes-agent:latest
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.docker.network=proxy
      - traefik.http.routers.hermes-agent.rule=Host(`hermes-agent.example.com`)
      - traefik.http.routers.hermes-agent.entrypoints=websecure
      - traefik.http.routers.hermes-agent.tls=true
      - traefik.http.routers.hermes-agent.tls.certresolver=mytlschallenge
      - traefik.http.services.hermes-agent.loadbalancer.server.port=4860
    env_file:
      - .env
    volumes:
      - ./data:/opt/data
    networks:
      - proxy

networks:
  proxy:
    external: true
```

Replace:

```text
hermes-agent.example.com
```

with your real Hermes hostname.

Do not publish Hermes publicly with:

```yaml
ports:
  - "4860:4860"
```

Traefik should expose Hermes through HTTPS and proxy to internal port `4860`.

### Hermes Environment

In the Hostinger environment UI or `.env`:

```env
ADMIN_USERNAME=your-admin-username
ADMIN_PASSWORD=your-admin-password
COMPOSE_PROJECT_NAME=hermes-agent
TRAEFIK_HOST=example.com
```

If your Traefik label hard-codes the full hostname, `COMPOSE_PROJECT_NAME` and `TRAEFIK_HOST` are not required for routing.

### Restart

From the Traefik stack folder:

```bash
docker compose up -d
```

From the Hermes stack folder:

```bash
docker compose up -d
```

Check:

```bash
docker ps
docker network inspect proxy
```

The `proxy` network should include both Traefik and Hermes.

## 2. Fix Hermes Data Permissions If Needed

Enter the Hermes app folder:

```bash
cd /docker/hermes-agent-XXXX
```

If Hermes cannot read `/opt/data`, fix ownership from the VPS:

```bash
docker compose exec -u root hermes-agent sh -lc 'chown -R hermes:hermes /opt/data && chmod -R u+rwX,go+rX /opt/data'
```

If the `hermes` user does not exist, use the numeric user that your image expects.

## 3. Connect Hermes To SuperGrok / xAI OAuth

Enter the Hermes container:

```bash
docker compose exec -it hermes-agent /bin/bash
```

Start xAI OAuth:

```bash
hermes auth add xai-oauth --no-browser
```

Hermes will print an authorization URL and wait for a callback on:

```text
http://127.0.0.1:56121/callback
```

Keep that terminal open.

### Capture The Callback On Windows

On your Windows PC, open PowerShell and run this before opening the xAI authorization URL:

```powershell
$tcp=[Net.Sockets.TcpListener]::new([Net.IPAddress]::Parse("127.0.0.1"),56121); $tcp.Start(); "Waiting for xAI callback..."; $c=$tcp.AcceptTcpClient(); $s=$c.GetStream(); $r=[IO.StreamReader]::new($s); $line=$r.ReadLine(); $path=($line -split " ")[1]; $url="http://127.0.0.1:56121$path"; while(($h=$r.ReadLine()) -ne ""){}; $body="Callback captured. You can close this tab."; $resp="HTTP/1.1 200 OK`r`nContent-Length: $($body.Length)`r`nConnection: close`r`n`r`n$body"; $b=[Text.Encoding]::UTF8.GetBytes($resp); $s.Write($b,0,$b.Length); $c.Close(); $tcp.Stop(); $url | Set-Clipboard; "COPIED CALLBACK URL:"; $url
```

Open the xAI authorization URL in your browser and approve it.

PowerShell should copy a callback URL like:

```text
http://127.0.0.1:56121/callback?code=...&state=...
```

### Relay The Callback Into The Container

Open a second Hermes container terminal:

```bash
cd /docker/hermes-agent-XXXX
docker compose exec -it hermes-agent /bin/bash
```

Paste the copied callback URL into `curl`:

```bash
curl 'http://127.0.0.1:56121/callback?code=PASTE_CODE_HERE&state=PASTE_STATE_HERE'
```

Use single quotes because the URL contains `&`.

The first terminal should complete the OAuth login.

### Configure Hermes Model

Inside the Hermes container:

```bash
hermes config set model.provider xai-oauth
hermes config set model.default grok-4.3
hermes config set model.base_url https://api.x.ai/v1
hermes auth list
hermes config show
hermes doctor
```

Test:

```bash
hermes -z "Say hello in one sentence." --provider xai-oauth -m grok-4.3
```

## 4. Create And Configure An X Developer App

Open the X Developer Console and create or edit an app.

Use:

```text
Environment: Production
App permissions: Read and write and Direct message
Type of App: Native App
Callback URI / Redirect URL: http://localhost:8080/callback
Website URL: https://hermes-agent.example.com
```

If the app asks for Terms of Service or Privacy Policy URLs, use real public pages. For a quick personal test, your public Hermes URL can satisfy the URL validator, but use proper policy pages for a public app.

If `xurl` requests `users.email`, enable `Request email from users` and provide valid Terms and Privacy URLs.

Save changes, then go to `Keys and tokens` and copy the OAuth 2.0 Client ID and Client Secret.

If you changed the app type, regenerate or recopy the OAuth 2.0 credentials.

## 5. Authenticate xurl On Windows First

The working path was to authenticate `xurl` locally on Windows, then copy the generated `.xurl` file into the VPS container.

Install Go if needed:

```powershell
winget install GoLang.Go
```

Close PowerShell, open a new PowerShell window, then check:

```powershell
go version
```

Install the Go-built `xurl` binary:

```powershell
go install github.com/xdevplatform/xurl@latest
& "$env:USERPROFILE\go\bin\xurl.exe" --version
```

Use the exact binary path so Windows does not pick up an older npm wrapper:

```powershell
& "$env:USERPROFILE\go\bin\xurl.exe" auth apps add YOUR_APP_NAME --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET --redirect-uri http://localhost:8080/callback
& "$env:USERPROFILE\go\bin\xurl.exe" auth default YOUR_APP_NAME
& "$env:USERPROFILE\go\bin\xurl.exe" auth oauth2 --app YOUR_APP_NAME
```

When the browser opens, the authorization URL should be long and include:

```text
redirect_uri=
scope=
state=
code_challenge=
code_challenge_method=S256
```

If the browser URL only shows `client_id=...`, your Windows machine is still using an old or broken `xurl` launcher.

After approval, verify locally:

```powershell
& "$env:USERPROFILE\go\bin\xurl.exe" auth status
& "$env:USERPROFILE\go\bin\xurl.exe" --auth oauth2 /2/users/me
```

## 6. Copy xurl Credentials To Hostinger Hermes

The local OAuth credentials are stored in:

```text
C:\Users\YOUR_WINDOWS_USER\.xurl
```

Copy it to the VPS:

```powershell
scp "$env:USERPROFILE\.xurl" root@YOUR_VPS_IP:/root/.xurl
```

SSH into the VPS:

```powershell
ssh root@YOUR_VPS_IP
```

Go to the Hermes compose folder:

```bash
cd /docker/hermes-agent-XXXX
```

Install `xurl` inside the Hermes container if it is not already installed:

```bash
docker compose exec hermes-agent sh -lc 'curl -fsSL https://raw.githubusercontent.com/xdevplatform/xurl/main/install.sh | bash'
```

The Hostinger Hermes ttyd terminal uses:

```text
HOME=/opt/data
```

So the working xurl credential file must be:

```text
/opt/data/.xurl
```

Copy the credential file into the container:

```bash
docker compose cp /root/.xurl hermes-agent:/opt/data/.xurl
docker compose exec -u root hermes-agent sh -lc 'chown hermes:hermes /opt/data/.xurl && chmod 600 /opt/data/.xurl'
```

Open the Hermes browser terminal and verify:

```bash
echo $HOME
which xurl
xurl auth status
xurl --auth oauth2 /2/users/me
```

User lookup:

```bash
xurl --auth oauth2 /2/users/by/username/YOUR_X_USERNAME
```

Prefer the raw OAuth2 endpoint above. Some `xurl` shortcuts may use a different auth mode and return `401 Unauthorized`.

## 7. X API Credits

OAuth can work even before you add credits, but real API requests can fail with:

```json
{
  "title": "CreditsDepleted",
  "detail": "Your enrolled account does not have any credits to fulfill this request."
}
```

Add the smallest credit amount the X Developer Console allows, then keep auto-recharge off while testing.

Low-cost verification commands:

```bash
xurl auth status
xurl --auth oauth2 /2/users/me
xurl --auth oauth2 /2/users/by/username/YOUR_X_USERNAME
```

Avoid broad searches, repeated loops, media uploads, direct message calls, and test posts with URLs while testing.

## Troubleshooting

### Gateway Timeout

If Hermes shows `Gateway Timeout`, Traefik can see the labels but cannot reach the Hermes container.

Check:

```bash
docker network inspect proxy
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Fix:

- Traefik and Hermes must both join `proxy`.
- Hermes labels must include `traefik.docker.network=proxy`.
- Only Traefik should publish `80` and `443`.
- Hermes should use the same cert resolver as Traefik, for example `mytlschallenge`.

### `curl -fsSL` Fails In PowerShell

Windows PowerShell aliases `curl` to `Invoke-WebRequest`. Use `curl.exe` or run the command on the Linux VPS:

```powershell
curl.exe -fsSL https://example.com/install.sh -o install.sh
```

### xurl Opens A Broken OAuth URL

If the browser opens only:

```text
https://x.com/i/oauth2/authorize?client_id=...
```

install and run the Go-built `xurl` binary by exact path:

```powershell
go install github.com/xdevplatform/xurl@latest
& "$env:USERPROFILE\go\bin\xurl.exe" auth oauth2 --app YOUR_APP_NAME
```

### xurl Opens OAuth Again In ttyd

Check which file `xurl` is reading:

```bash
echo $HOME
ls -la "$HOME/.xurl"
xurl auth status
```

For Hostinger Hermes ttyd, the working file should be:

```text
/opt/data/.xurl
```

If the token was copied to `/opt/data/home/.xurl`, copy it into `/opt/data/.xurl`:

```bash
cp /opt/data/home/.xurl /opt/data/.xurl
chmod 600 /opt/data/.xurl
```

If ownership is wrong, fix it from the VPS host:

```bash
docker compose exec -u root hermes-agent sh -lc 'chown hermes:hermes /opt/data/.xurl && chmod 600 /opt/data/.xurl'
```

### `xurl user USERNAME` Returns 401

Use OAuth2 explicitly:

```bash
xurl --auth oauth2 /2/users/by/username/YOUR_X_USERNAME
```

## References

- Hermes Agent repository: https://github.com/NousResearch/hermes-agent
- Hostinger VPS: https://www.hostinger.com/vps
- Traefik Docker provider: https://doc.traefik.io/traefik/providers/docker/
- Traefik Let's Encrypt / ACME: https://doc.traefik.io/traefik/https/acme/
- Docker Compose networking: https://docs.docker.com/compose/how-tos/networking/
- xAI docs: https://docs.x.ai/
- xAI models: https://docs.x.ai/developers/models
- xAI image generation: https://docs.x.ai/docs/guides/image-generation
- xAI video generation: https://docs.x.ai/docs/guides/video-generations
- xAI voice / TTS: https://docs.x.ai/docs/guides/voice
- X xurl docs: https://docs.x.com/tools/xurl
- xurl GitHub: https://github.com/xdevplatform/xurl
- X OAuth 2.0 Authorization Code Flow with PKCE: https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code
- X Developer apps: https://docs.x.com/fundamentals/developer-apps
- X API pricing: https://docs.x.com/x-api/getting-started/pricing
