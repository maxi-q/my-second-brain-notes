# Развёртывание my-second-brain MCP на новом сервере

Дата: 2026-10-26

Новый сервер: `ssh proclaude`. Старый сервер: `ssh imanator.pro`.

MCP URL: https://sb-mcp.dev.imanator.pro/mcp

**Локально — PowerShell, папка `my-second-brain`**

```powershell
git remote add origin https://github.com/maxi-q/my-second-brain-mcp.git
git fetch origin
git switch -c main --track origin/main

Get-Content -LiteralPath .env.docker.example,Dockerfile,.dockerignore,.gitignore
git status --short --branch
```

**Новый сервер — `ssh proclaude`**

```bash
hostname
cat /etc/os-release
id
command -v docker
docker compose version
command -v nginx
command -v caddy
ls -ld /opt/*brain* /srv/*brain* /root/*brain* 2>/dev/null
ss -ltnp
```

**Локально — PowerShell**

```powershell
Resolve-DnsName sb-mcp.dev.imanator.pro -Type A
Resolve-DnsName sb-mcp.dev.imanator.pro -Type AAAA -ErrorAction SilentlyContinue
```

**Новый сервер — `proclaude`**

```bash
find /root/second-brain-plugin -maxdepth 2 -type f -printf "%P\n"
docker ps --format "{{.Names}} {{.Image}} {{.Ports}}"
ls -l /etc/nginx/sites-enabled /etc/nginx/conf.d
grep -R -l sb-mcp /etc/nginx /etc/letsencrypt/renewal 2>/dev/null
ls -la /root/.ssh

find /opt /srv /root -maxdepth 3 -type f \
  \( -name "*.env" -o -name ".env" -o -name "*brain*" \) \
  -printf "%p\n" 2>/dev/null

command -v certbot
ls -l /etc/letsencrypt/live 2>/dev/null
cat /etc/nginx/sites-available/default
cat /etc/nginx/sites-available/ydb.aheron.pro
df -h /
free -m
git --version
curl -Is --max-time 15 https://api.github.com/meta | head -n 1
```

**Локально — PowerShell**

```powershell
$taskArchivePath = Join-Path $env:TEMP 'second-brain-mcp-309e6f4.tar'
git archive --format=tar --output=$taskArchivePath HEAD
scp $taskArchivePath proclaude:/tmp/
```

**Новый сервер — `proclaude`**

```bash
set -eu

app=/root/second-brain-plugin
cd "$app"
test ! -e server.py
tar -xf /tmp/second-brain-mcp-309e6f4.tar

mkdir -p data secrets
chmod 700 secrets
chown 10001:10001 data

if [ ! -e secrets/git_key ]; then
  ssh-keygen -q -t ed25519 \
    -f secrets/git_key \
    -N '' \
    -C 'second-brain-mcp-proclaude-2026-10-26'
fi

curl --fail --silent --show-error https://api.github.com/meta |
  python3 -c 'import json,sys; m=json.load(sys.stdin); print("\n".join("github.com "+k for k in m["ssh_keys"]))' \
  > secrets/known_hosts

chown 10001:10001 secrets/git_key
chmod 400 secrets/git_key
chmod 644 secrets/known_hosts

cat secrets/git_key.pub

docker build -t second-brain-mcp:local .
```

**Локально — PowerShell, добавление deploy key в GitHub**

```powershell
$taskCredentialRequest = "protocol=https`nhost=github.com`npath=maxi-q/my-second-brain-mcp.git`n`n"

$taskCredentialLines = $taskCredentialRequest |
  git -c credential.interactive=never credential fill 2>$null

if ($LASTEXITCODE -ne 0) {
  throw 'GitHub credential unavailable without interactive login'
}

$taskCredentialFields = @{}

foreach ($taskCredentialLine in $taskCredentialLines) {
  $taskPair = $taskCredentialLine -split '=',2
  if ($taskPair.Count -eq 2) {
    $taskCredentialFields[$taskPair[0]] = $taskPair[1]
  }
}

if (-not $taskCredentialFields['password']) {
  throw 'No GitHub credential available'
}

$taskHeaders = @{
  Authorization = 'Bearer ' + $taskCredentialFields['password']
  Accept = 'application/vnd.github+json'
  'X-GitHub-Api-Version' = '2022-11-28'
}

$taskPublicKey = (
  ssh -o BatchMode=yes proclaude `
    'cat /root/second-brain-plugin/secrets/git_key.pub'
).Trim()

if ($taskPublicKey -notmatch '^ssh-ed25519 AAAA') {
  throw 'Unexpected public key format'
}

$taskKeysUri = 'https://api.github.com/repos/maxi-q/my-second-brain-notes/keys'

$taskExistingKeys = Invoke-RestMethod `
  -Uri $taskKeysUri `
  -Headers $taskHeaders `
  -Method Get

$taskMatch = @(
  $taskExistingKeys |
    Where-Object { $taskPublicKey.StartsWith($_.key) }
)

if ($taskMatch.Count -eq 0) {
  $taskKeyPayload = @{
    title = 'second-brain-mcp-proclaude-2026-10-26'
    key = $taskPublicKey
    read_only = $false
  } | ConvertTo-Json

  $taskCreatedKey = Invoke-RestMethod `
    -Uri $taskKeysUri `
    -Headers $taskHeaders `
    -Method Post `
    -Body $taskKeyPayload `
    -ContentType 'application/json'

  Write-Output (
    'Deploy key added; id=' + $taskCreatedKey.id +
    '; read_only=' + $taskCreatedKey.read_only
  )
}

$taskCredentialFields.Clear()
$taskHeaders.Clear()
```

**Новый сервер — `proclaude`, Nginx и сертификат**

```bash
set -eu

site=/etc/nginx/sites-available/sb-mcp.dev.imanator.pro
test ! -e "$site"

cat > "$site" <<'NGINX'
server {
    listen 80;
    listen [::]:80;
    server_name sb-mcp.dev.imanator.pro;
    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 503; }
}
NGINX

ln -s "$site" /etc/nginx/sites-enabled/sb-mcp.dev.imanator.pro

nginx -t
systemctl reload nginx

certbot certonly \
  --webroot \
  -w /var/www/html \
  -d sb-mcp.dev.imanator.pro \
  --non-interactive \
  --keep-until-expiring

cat > "$site" <<'NGINX'
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name sb-mcp.dev.imanator.pro;
    ssl_certificate /etc/letsencrypt/live/sb-mcp.dev.imanator.pro/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sb-mcp.dev.imanator.pro/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    server_tokens off;
    access_log /var/log/nginx/sb-mcp.dev.imanator.pro.access.log;
    error_log /var/log/nginx/sb-mcp.dev.imanator.pro.error.log;
    location / {
        client_max_body_size 1m;
        proxy_pass http://127.0.0.1:18000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_read_timeout 180s;
    }
}
server {
    listen 80;
    listen [::]:80;
    server_name sb-mcp.dev.imanator.pro;
    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 301 https://sb-mcp.dev.imanator.pro$request_uri; }
}
NGINX

nginx -t
systemctl reload nginx
systemctl is-active certbot.timer
```

**Новый сервер — `proclaude`, репозиторий заметок и тесты**

```bash
set -eu
cd /root/second-brain-plugin

if [ ! -e data/repo ]; then
  docker run --rm \
    --read-only \
    --cap-drop ALL \
    --security-opt no-new-privileges:true \
    --tmpfs /tmp:size=64m,mode=1777 \
    -v "$PWD/data:/data" \
    -v "$PWD/secrets/git_key:/run/secrets/git_key:ro" \
    -v "$PWD/secrets/known_hosts:/run/secrets/git_known_hosts:ro" \
    -e GIT_TERMINAL_PROMPT=0 \
    -e 'GIT_SSH_COMMAND=ssh -i /run/secrets/git_key -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=/run/secrets/git_known_hosts' \
    second-brain-mcp:local \
    git clone --branch main \
    git@github.com:maxi-q/my-second-brain-notes.git /data/repo
fi

docker run --rm \
  -v "$PWD/data:/data" \
  second-brain-mcp:local \
  git -C /data/repo config user.name 'Second Brain MCP'

docker run --rm \
  -v "$PWD/data:/data" \
  second-brain-mcp:local \
  git -C /data/repo config user.email \
  'second-brain-mcp@users.noreply.github.com'

docker run --rm \
  -v "$PWD/test_server.py:/app/test_server.py:ro" \
  second-brain-mcp:local \
  python -m unittest -v test_server

printf '%s\n' 309e6f439820ea0d561f9d45fe58aaa1465a363f \
  > DEPLOYED_REVISION
```

**Новый сервер — `proclaude`, проверка SSH GitHub**

```bash
ps -eo pid,etime,args |
  grep -E "(git clone|ssh -i /run/secrets|unittest)" |
  grep -v grep

timeout 15 ssh \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  -o UserKnownHostsFile=/root/second-brain-plugin/secrets/known_hosts \
  -i /root/second-brain-plugin/secrets/git_key \
  -T git@github.com

timeout 15 ssh \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  -o HostKeyAlias=github.com \
  -o UserKnownHostsFile=/root/second-brain-plugin/secrets/known_hosts \
  -i /root/second-brain-plugin/secrets/git_key \
  -T -p 443 git@ssh.github.com
```

**Старый сервер — `ssh imanator.pro`**

```bash
hostname
id
ls -la ~/my-second-brain

find ~/my-second-brain -maxdepth 3 -type f \
  \( -name .env -o -name "*.env" \
     -o -name compose.yaml -o -name compose.yml \
     -o -name docker-compose.yml -o -name AGENTS.md \) \
  -printf "%p\n"

docker ps --format "{{.Names}} {{.Image}}"

cd ~/my-second-brain/my-second-brain-mcp

grep -E \
  "^(PUBLIC_URL|OAUTH_ISSUER|OAUTH_JWKS_URL|ALLOWED_SUB|NOTES_BRANCH)=" \
  .env

git status --short --branch
git rev-parse HEAD

docker compose exec -T mcp git -C /data/repo status --short --branch
docker compose exec -T mcp git -C /data/repo log -1 --format="%h %ad"
docker compose exec -T mcp git -C /data/repo rev-list --left-right --count origin/main...HEAD
docker compose exec -T mcp git -C /data/repo fetch origin
docker compose exec -T mcp git -C /data/repo rev-list --left-right --count origin/main...HEAD
```

**Локально — PowerShell, перенос настроек**

```powershell
$taskEnvLines = @(
  ssh -o BatchMode=yes imanator.pro `
    'grep -E "^(PUBLIC_URL|OAUTH_ISSUER|OAUTH_JWKS_URL|ALLOWED_SUB|NOTES_BRANCH)=" ~/my-second-brain/my-second-brain-mcp/.env'
)

if ($LASTEXITCODE -ne 0 -or $taskEnvLines.Count -ne 5) {
  throw 'Incomplete OAuth configuration'
}

if ($taskEnvLines -notcontains 'PUBLIC_URL=https://sb-mcp.dev.imanator.pro/mcp') {
  throw 'Unexpected public URL'
}

$taskEnvLines |
  ssh -o BatchMode=yes proclaude `
    'umask 077; set -C; cat > /root/second-brain-plugin/.env'
```

**Старый сервер — `imanator.pro`**

```bash
cd ~/my-second-brain/my-second-brain-mcp

docker compose stop mcp

git -c safe.directory="$PWD/data/repo" \
  -C data/repo status --short --branch

git -c safe.directory="$PWD/data/repo" \
  -C data/repo rev-parse HEAD
```

**Новый сервер — `proclaude`**

```bash
cd /root/second-brain-plugin

docker compose up -d --no-build
docker compose ps
docker compose logs --tail=25 mcp

docker compose exec -T mcp \
  git -C /data/repo merge-base --is-ancestor \
  f707830f19ae9588c7b8d3b22fb269274916dc8e HEAD

docker compose exec -T mcp \
  git -C /data/repo push --dry-run origin HEAD:refs/heads/main

curl -fsS \
  http://127.0.0.1:18000/.well-known/oauth-protected-resource/mcp

curl -sS -o /dev/null \
  -w "\nPUBLIC_DISCOVERY_HTTP=%{http_code}\n" \
  https://sb-mcp.dev.imanator.pro/.well-known/oauth-protected-resource/mcp

curl -sS -D - -o /dev/null \
  -X POST https://sb-mcp.dev.imanator.pro/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data-binary "{}"
```

**Новый сервер — `proclaude`, заключительные проверки**

```bash
set -eu
cd /root/second-brain-plugin

docker compose exec -T mcp \
  git -C /data/repo merge-base --is-ancestor \
  f707830f19ae9588c7b8d3b22fb269274916dc8e HEAD

docker compose exec -T mcp \
  git -C /data/repo status --short --branch

curl -fsS \
  https://dev-ayw2n8j5l6wvn17e.us.auth0.com/.well-known/openid-configuration \
  -o /dev/null

curl -fsS \
  https://dev-ayw2n8j5l6wvn17e.us.auth0.com/.well-known/jwks.json \
  -o /dev/null

docker inspect \
  --format "status={{.State.Status}} health={{.State.Health.Status}} restart={{.HostConfig.RestartPolicy.Name}}" \
  second-brain-mcp-mcp-1

stat -c "%a %n" .env secrets/git_key
```

**Старый сервер — `imanator.pro`**

```bash
docker inspect \
  --format "old_status={{.State.Status}}" \
  second-brain-mcp-mcp-1
```
