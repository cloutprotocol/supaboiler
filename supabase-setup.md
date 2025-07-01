# Self-Hosted Supabase Deployment – July 2025

> **Server**  `37.27.224.180` (Ubuntu 24.04 LTS, root on RAID-1 `/dev/md2`)
>
> **Stack root**  `/opt/supabase-min`
>
> **Primary entry points**
> * API gateway (Kong): http://37.27.224.180:4269
> * Studio (GUI):      http://37.27.224.180:4369

---

## 1. Directory Layout

```text
/opt/supabase-min/
├─ docker-compose.yml      # minimal stack definition (services below)
├─ .env                    # **private** secrets copied from `.env-temp`
├─ kong.yml                # declarative Kong routes
└─ volumes/
   ├─ db/data              # postgres data → bind-mounted to /var/lib/supabase/data
   └─ storage/files        # local file backend for Storage API
```

> All data lies on the RAID-1 array; the NVMe volume used by Bitcoin/Ordinals is **not** touched.

---

## 2. Running Services

| Service  | Image/tag                                 | Ports (host → cont.) | Path behind Kong |
|----------|-------------------------------------------|----------------------|------------------|
| `db`     | `supabase/postgres:15.1.0.11`             | —                    | —                |
| `kong`   | `kong:2.8.1`                              | 4269 → 8000          | —                |
| `rest`   | `supabase/postgrest:v11.2.2`              | —                    | `/rest/v1/*`     |
| `auth`   | `supabase/gotrue:v2.176.1`                | —                    | `/auth/v1/*`     |
| `storage`| `supabase/storage-api:v0.46.6`            | —                    | `/storage/v1/*`  |
| `realtime`* | `supabase/realtime:v2.24.1`            | —                    | `/realtime/v1/socket` |
| `studio` | `supabase/studio:latest`                  | 4369 → 3000          | *(direct port)*  |

\* Realtime boots with   `DB_NAME=postgres  PORT=4000`  (added in compose).

---

## 3. Secrets & Environment

All secrets live in `.env` and are injected by Compose. **Do not commit real values**; rotate before production.

```env
POSTGRES_PASSWORD=…
JWT_SECRET=…
ANON_KEY=…
SERVICE_ROLE_KEY=…
DASHBOARD_USERNAME=supabase
DASHBOARD_PASSWORD=Igetrich12345
SITE_URL=http://37.27.224.180:4269
SUPABASE_PUBLIC_URL=${SITE_URL}
# …see full file for the rest
```

Studio's basic auth uses `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD`.

---

## 4. Lifecycle Commands

```bash
# start / stop everything
cd /opt/supabase-min
docker compose up -d           # start
docker compose stop            # stop (preserves data)

# start only Studio
docker compose up -d studio

# tail logs for a service
docker logs -f supabase-min-<service>-1
```

---

## 5. Firewall

Hetzner Cloud firewall rules added:

* **TCP 4269** – Kong (API)
* **TCP 4369** – Studio (GUI)

Everything else remains internal to the Docker network.

---

## 6. Actions Completed

- [x] Cloned minimal Compose files into `/opt/supabase-min`.
- [x] Populated `.env` with generated secrets.
- [x] Bound Postgres data to `/var/lib/supabase/data` on RAID-1.
- [x] Deployed core services (`db`, `kong`, `rest`, `auth`, `storage`).
- [x] Fixed YAML/indent errors and env wiring.
- [x] Provisioned `storage/files` directory with UID 1000 ownership.
- [x] Enabled `realtime` service with minimal env.
- [x] Added `studio` container exposed on port 4369.
- [x] Opened corresponding Hetzner firewall ports.

---

## 7. TODO / Next Steps

- [ ] **TLS** – terminate with Nginx/Traefik or Let's Encrypt on ports 80/443 and forward to Kong.
- [ ] **Kong routing for Studio** – optional: proxy `/studio/*` through Kong to drop public port 4369.
- [ ] **Backup strategy**
  - Automated `pg_dump` + WAL shipping.
  - Snapshot `volumes/storage/files`.
- [ ] **Monitoring & Alerts** – integrate Vector/Logflare or external Prometheus.
- [ ] **Secret rotation** – replace current placeholder secrets before production.
- [ ] **Automated updates** – pin image tags and use Renovate or similar.
- [ ] **CI/CD** – wire up migrations + branch preview (Supabase CLI) if desired.

---

### Reference: Minimal `docker-compose.yml` snippet for Studio

```yaml
  studio:
    image: supabase/studio:latest
    depends_on:
      - rest
    ports:
      - "4369:3000"
    environment:
      STUDIO_PG_META_URL: http://rest:3000
      SUPABASE_URL: http://kong:8000
      SUPABASE_ANON_KEY: ${ANON_KEY}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      DASHBOARD_USERNAME: ${DASHBOARD_USERNAME}
      DASHBOARD_PASSWORD: ${DASHBOARD_PASSWORD}
```

---

## 8. Boilerplate Template – Coolify-ready Supabase Stack

> This template lives under `supabase/` in the repo.  Copy the directory, tweak two items, and you have a fresh self-hosted Supabase (plus optional OCR service) ready for Coolify.

### 8.1  Directory snapshot

```text
supabase/
├─ docker-compose.yml     # the stack definition (db, kong, rest … ocr)
├─ env.example            # copy → .env and fill secrets
└─ volumes/               # empty – host-bound at runtime
```

### 8.2  One-time setup per **new** project

1. `cp -r supabase <my-project>` – gives you an isolated Compose bundle.
2. Choose an unused *base port* **P** (e.g. 50 00, 50 10, …).  Edit `<my-project>/docker-compose.yml`:
   * `kong` → `${P}:8000`
   * `studio` → `${P}+100:3000`
3. Copy `env.example` → `.env` and generate fresh values *(see §3 above)*.
4. On the host **before** Coolify deploys, open the two ports:

   ```bash
   sudo ufw allow ${P}/tcp        # Kong (API)
   sudo ufw allow $((P+100))/tcp  # Studio (GUI)
   ```

5. Push branch → Coolify → **Add > Docker-Compose**; set *Working Directory* to `<my-project>`.
6. Hit *Deploy* – Coolify will pull images and start the stack.

### 8.3  Default service matrix

| Service   | Port (container) | Example host port | Notes |
|-----------|------------------|-------------------|-------|
| kong      | 8000             | **P**             | API gateway |
| studio    | 3000             | **P** + 100        | optional; may be proxied via Kong |
| db        | internal         | —                 | data lives in `volumes/db/data` |
| rest      | 3000             | —                 | behind Kong `/rest/*` |
| auth      | 9999             | —                 | behind Kong `/auth/*` |
| storage   | 5000             | —                 | behind Kong `/storage/*` |
| realtime  | 4000             | —                 | behind Kong `/realtime/*` |
| ocr¹      | 7000             | —                 | REST endpoint for OCR |

¹ Image + config are stubs; swap for your preferred OCR container.

### 8.4  Adding Prisma-OCR service

The Compose file ships with a commented `ocr:` service using a generic Tesseract-based image.  Uncomment and adapt environment / volumes when you actually need OCR.  Supabase functions (or any backend) can then call the OCR endpoint within the Compose network at `http://ocr:7000`.

### 8.5  Port-planning quick reference

| Instance # | Kong host port (**P**) | Studio host port (**P+100**) |
|-----------:|-----------------------:|-----------------------------:|
| 1          | 4269                  | 4369                        |
| 2          | 4270                  | 4370                        |
| 3          | 4271                  | 4371                        |
| 4          | 4272                  | 4372                        |
| …          | …                     | …                           |

> Rule of thumb: pick any unused base port **P** ≥ 4269.  Reserve **P+100** for the matching Studio container.  Only these two ports need UFW rules; all other Supabase services stay internal.

---

End of new boilerplate section – keep in sync with the files under `supabase/`.
