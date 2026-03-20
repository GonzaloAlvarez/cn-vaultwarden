# cn-vaultwarden

Vaultwarden on a Raspberry Pi, exposed two ways:

- **`https://passwords.<LAB_DOMAIN>`** — accessible from the tailnet via traefik-lab on the VPS
- **`https://passwords.lan`** (or your `LAN_DOMAIN`) — accessible on the local LAN via a self-signed cert

## Services

| Service | Description |
|---|---|
| `ts-vault` | Tailscale sidecar — joins the tailnet as `vault.<TAILNET_DOMAIN>` |
| `vaultwarden` | Password manager, runs in `ts-vault`'s network namespace |
| `consul-register` | Registers Vaultwarden with the VPS Consul every 60s, enabling traefik-lab to route to it |
| `traefik-lan` | Reverse proxy on the Pi's LAN interface — terminates TLS for `passwords.lan` |

## Before you start

### 1. On the VPS — create a pre-auth key with `tag:svc`

```sh
docker exec cloudnet-headscale-1 headscale preauthkeys create --user 1 --tags tag:svc --reusable --expiration 1h
```

Copy the key — this is your `VAULT_AUTHKEY`.

### 2. On the VPS — verify the ACL allows `tag:svc → tag:infra:8500`

The Pi needs to reach the VPS Consul on port 8500. Check `headscale/acl.hujson` in `cn-root-docker` includes:

```json
{ "action": "accept", "src": ["tag:svc"], "dst": ["tag:infra:8500"] }
```


## Setup

### 1. Create `.env`

```sh
./setup.sh          # production (default)
./setup.sh staging  # if the VPS is still using Let's Encrypt staging certs
```

The script prompts for each variable, generates a self-signed TLS cert for the LAN domain, and generates config files from templates.

### 2. Variable reference

| Variable | Where to get it |
|---|---|
| `ROOT_DOMAIN` | Your apex domain (e.g. `gn.al`) |
| `LAB_DOMAIN` | Internal lab domain (e.g. `lab.gn.al`) |
| `TAILNET_DOMAIN` | MagicDNS base domain (e.g. `ts.gn.al`) |
| `HEADSCALE_DOMAIN` | Headscale public hostname (e.g. `hs.gn.al`) |
| `VAULT_AUTHKEY` | Pre-auth key created in step above with `--tags tag:svc` |
| `VAULTWARDEN_ADMIN_TOKEN` | Token for the Vaultwarden admin panel at `/admin`. Generate a secure token: `openssl rand -base64 48` |
| `PI_LAN_IP` | Pi's LAN IP address (e.g. `192.168.1.42`) |
| `LAN_DOMAIN` | LAN hostname — defaults to `passwords.lan` |
| `SMTP_*` | Optional — leave blank to disable email |

### 3. Start services

```sh
docker compose up -d
```

### 4. Install the LAN certificate (optional)

To avoid browser certificate warnings on the LAN:

- `certs/lan.crt` is the self-signed CA/certificate for `passwords.lan`
- Install it as a **trusted root CA** on each device:
  - **macOS**: Keychain Access → drag in `lan.crt` → mark as "Always Trust"
  - **iOS**: Settings → General → VPN & Device Management → install profile, then Settings → General → About → Certificate Trust Settings → enable full trust
  - **Android**: Settings → Security → Install from storage

## Bring-up sequence

After `docker compose up -d`:

1. Check `ts-vault` joins the tailnet: `docker compose logs ts-vault`
2. Verify Consul registration: the `consul-register` container logs `Registered vaultwarden-vault` every 60s
3. Access `https://passwords.lab.gn.al` from a tailnet device
4. Access `https://passwords.lan` from a LAN device

## Switching between staging and production

When the VPS switches from Let's Encrypt staging to production certs:

```sh
./setup.sh          # switches to production mode, removes staging root certs
docker compose down && docker compose up -d
```
