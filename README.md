# Supaboiler – Self-Hosted Supabase Boilerplate

Minimal Docker-Compose stack covering Postgres, Kong, Auth, REST, Storage, Realtime and Studio.

---

## Quick start (local)

```bash
# clone and enter repo
cp env.example .env            # generate fresh secrets first!
# optional: override default ports
# echo "KONG_HOST_PORT=4270" >> .env
# echo "STUDIO_HOST_PORT=4370" >> .env
docker compose pull            # fetch images
docker compose up -d           # launch stack
```

Access:
* Kong gateway →  http://localhost:${KONG_HOST_PORT:-4269}
* Studio (GUI) → http://localhost:${STUDIO_HOST_PORT:-4369}

---

## Deploy on Coolify
1. Add this repo as a *Docker-Compose* app.
2. In **Environment Variables** set at least:
   * `POSTGRES_PASSWORD`, `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`
   * `KONG_HOST_PORT`, `STUDIO_HOST_PORT` (choose an unused pair)
3. Open the two ports on your server firewall.
4. Hit *Deploy* – Coolify will pull the images and start the stack.

---

## Notes
* `rest` now uses the official image `postgrest/postgrest:v11.2.2` – no auth or rate-limit issues when pulling.
* All data is stored inside the `volumes/` directory; keep it out of Git (see `.gitignore`).
* For a production-ready setup, terminate TLS in front of Kong and enable a backup strategy. 