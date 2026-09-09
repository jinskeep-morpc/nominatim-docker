# Serving Nominatim through a Cloudflare Tunnel with a service token

This runs the Nominatim container on a VPS with **no public ports**. Traffic reaches
it only through an outbound Cloudflare Tunnel, and Cloudflare Access rejects any
request that does not present a valid **service token** at the edge — before it
ever touches the VPS.

```
client ──(CF-Access-Client-Id / CF-Access-Client-Secret)──► Cloudflare edge
        └─ Access policy: allow only service token ─┘
                                    │
                              cloudflared (outbound tunnel, no inbound ports)
                                    │  private docker network "edge"
                                    ▼
                              nominatim:8080
```

## 1. VPS prep

```bash
# firewall: only SSH inbound; the tunnel is outbound so nothing else is needed
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable

sudo mkdir -p /data/db /data/flatnode
```

## 2. Secrets

```bash
cd contrib
cp .env.example .env
# NOMINATIM_PASSWORD: openssl rand -hex 24
# CF_TUNNEL_TOKEN:    filled in step 3
$EDITOR .env
```

`contrib/.env` is gitignored — keep it that way.

## 3. Create the tunnel (Cloudflare dashboard)

1. **Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared.**
2. Name it (e.g. `nominatim-prod`). Copy the **tunnel token** into `CF_TUNNEL_TOKEN`
   in `contrib/.env`. (Ignore the install snippet — compose runs cloudflared for you.)
3. **Public Hostnames → Add a public hostname:**
   - Subdomain / domain: `nominatim.example.com`
   - Service: `HTTP` → `nominatim:8080`
     (container name + port; both containers share the `edge` network)
4. Save. DNS is created automatically.

## 4. Create the service token

**Zero Trust → Access → Service Auth → Create Service Token.**

- Name: e.g. `nominatim-client-1` (make one per consumer so you can revoke individually).
- Copy **Client ID** and **Client Secret** now — the secret is shown only once.
- Default expiry is 1 year; set a calendar reminder to rotate.

## 5. Protect the hostname with an Access application

**Zero Trust → Access → Applications → Add an application → Self-hosted.**

- Application domain: `nominatim.example.com`
- Session duration: any (irrelevant for tokens)
- **Policy:**
  - Action: **Service Auth**
  - Rule: **Include → Service Token → `nominatim-client-1`**
    (or *Any Access Service Token* to accept every token you issue)
- Save.

> Action must be **Service Auth**, not Allow. "Allow" would still let interactive
> (browser SSO) sessions through; "Service Auth" accepts *only* the token headers.

All callers here are machine-to-machine, so no CORS/`OPTIONS` bypass is needed.

## 6. Bring it up

```bash
cd contrib
docker compose up -d
docker compose logs -f nominatim   # first run imports the PBF; takes a while
```

`cloudflared` will show `Registered tunnel connection` once linked.

## 7. Test

```bash
# no token -> 403 from Cloudflare
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://nominatim.example.com/search?q=columbus&format=json"

# with token -> 200
curl -s "https://nominatim.example.com/search?q=columbus&format=json" \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>"
```

Service tokens are **header-only** — they cannot be passed as a query parameter.

## 8. Recommended extra: rate limiting

Even with a valid token, a client can hammer the database. Add a WAF rate-limiting
rule (**Security → WAF → Rate limiting rules**):

- If incoming requests match `Hostname eq nominatim.example.com`
- When rate exceeds e.g. `10` requests per `10s` per client IP
- Then **Block** for `60s`

## Rotating / revoking

- **Revoke one consumer:** delete its service token (Service Auth). Others keep working.
- **Rotate a secret:** create a new token, update the consumer, delete the old token.
- **Rotate the tunnel:** create a new tunnel, swap `CF_TUNNEL_TOKEN`, `docker compose up -d cloudflared`, delete the old tunnel.

## Notes

- Nothing is published on the host (`docker compose ps` shows no port mappings).
  `curl http://<vps-ip>:8080` from outside must fail.
- To reach Nominatim locally on the VPS for debugging:
  `docker compose exec nominatim curl -s localhost:8080/status`
- The `/status` endpoint is behind the same Access policy; use it via the tunnel
  with the token, or locally as above, for health checks.
