# Railway Supabase template update checklist

This checklist includes only changes needed beyond the supplied Railway template. Items already present in the template are intentionally omitted unless they need to be renamed, removed, or changed. It also accounts for the Railway-specific `github.com/6ixfalls/supabase` source repository, which already contains custom Kong, Postgres, and Supavisor assets.

## Railway source repository findings

The `6ixfalls/supabase` repository already has Railway-specific source folders for `kong`, `postgres`, and `pooler`, so the template should update those assets rather than recreate them from scratch:

- `kong/Dockerfile` currently builds from `kong:2.8.1`, copies `kong.yml`, installs `gettext`, and renders the Kong template with `envsubst` at startup.
- `kong/kong.yml` already contains the existing Auth, REST, GraphQL, Realtime, Analytics, postgres-meta, and Studio routes, but its Storage and Edge Functions routes are commented out.
- `postgres/Dockerfile` already bakes the Supabase init SQL files into `/docker-entrypoint-initdb.d` and installs a Railway-specific `wrapper.sh`.
- `postgres/wrapper.sh` already handles Railway-specific `PGHOST`/`PGPORT` behavior and persists `/etc/postgresql-custom` through the Postgres data volume.
- `pooler/Dockerfile` already bakes `pooler.exs` into a Supavisor image and starts Supavisor with migrate, eval, and server commands.

## Credential generation answer

The legacy symmetric API keys are **deterministic once their inputs are chosen**:

- Choose one `JWT_SECRET` with at least 32 characters.
- Sign an `anon` JWT with that secret and stable claims such as `role=anon`, `iss=supabase`, `iat`, and `exp`.
- Sign a `service_role` JWT with the same secret and stable claims such as `role=service_role`, `iss=supabase`, `iat`, and `exp`.
- Reusing the same secret, algorithm, header, and claims produces the same JWT string.
- Changing any input, including `iat` or `exp`, produces a different JWT string.

Railway can generate random strings, but it cannot automatically derive the signed Supabase JWT API keys from a shared `JWT_SECRET` or enforce every required key length/format. A small secret generator website is therefore useful for one-click template users. Build it as a static, client-only utility that uses browser cryptography, avoids telemetry, never sends generated values to a server, and clearly warns users to save credentials before closing the page.

The generator should produce Railway-ready variable values for each deployment:

- `JWT_SECRET`
- `ANON_KEY`
- `SERVICE_ROLE_KEY`
- `DASHBOARD_USERNAME`
- `DASHBOARD_PASSWORD`
- `MINIO_ROOT_USER`
- `MINIO_ROOT_PASSWORD`
- `SECRET_KEY_BASE`
- `REALTIME_DB_ENC_KEY`
- `VAULT_ENC_KEY`
- `PG_META_CRYPTO_KEY`
- `S3_PROTOCOL_ACCESS_KEY_ID`
- `S3_PROTOCOL_ACCESS_KEY_SECRET`
- Optional modern auth keys: `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY`, `JWT_KEYS`, `JWT_JWKS`, `ANON_KEY_ASYMMETRIC`, and `SERVICE_ROLE_KEY_ASYMMETRIC`

## Secret generator website requirements

- [x] Build a small static website or single-page tool for generating the template credentials at `docs/supabase-credential-generator.html`.
- [x] Run all generation locally in the browser; do not send generated secrets to any backend.
- [x] Use `crypto.getRandomValues` or WebCrypto APIs for random bytes.
- [x] Generate `JWT_SECRET` first, then sign `ANON_KEY` and `SERVICE_ROLE_KEY` from that same secret.
- [x] Use stable JWT claims so users can regenerate the same keys when they provide the same `JWT_SECRET`, `iat`, and `exp` inputs.
- [x] Provide defaults for `iat` and a long-lived `exp`, but make both visible so users understand they affect deterministic output.
- [ ] Validate `JWT_SECRET` is at least 32 characters.
- [ ] Validate `REALTIME_DB_ENC_KEY` is exactly 16 characters.
- [ ] Validate `VAULT_ENC_KEY` is exactly 32 characters.
- [ ] Validate `PG_META_CRYPTO_KEY` is at least 32 characters.
- [ ] Validate `SECRET_KEY_BASE` is at least 64 characters.
- [x] Generate MinIO and S3 protocol credentials instead of relying on hard-coded template defaults.
- [x] Offer a copyable Railway variable block with one `KEY=value` line per credential.
- [ ] Offer individual copy buttons for each credential, if the basic copy-all workflow is not enough.
- [x] Add a warning that `SERVICE_ROLE_KEY`, `SUPABASE_SECRET_KEY`, database passwords, and S3 secrets must never be exposed in browser application code.
- [x] Avoid analytics, external scripts, remote fonts, and third-party assets on the generator page.
- [x] Make the page usable offline after load by keeping CSS and JavaScript on the same page.

## 1. Update image sources and versions

- [ ] Change Studio from `supabase/studio:2025.10.09-sha-433e578` to `supabase/studio:2026.06.03-sha-0bca601`.
- [ ] Keep the Railway-specific Kong source repository, but update `kong/Dockerfile` from `kong:2.8.1` to `kong/kong:3.9.1`.
- [ ] Keep the existing `envsubst` startup pattern unless Railway can mount the official entrypoint and declarative config directly.
- [ ] Update `kong/kong.yml` with the new upstream routes/plugins instead of replacing the whole Railway template blindly.
- [ ] Change Auth from `supabase/gotrue:v2.180.0` to `supabase/gotrue:v2.189.0`.
- [ ] Change PostgREST from `postgrest/postgrest:v13.0.7` to `postgrest/postgrest:v14.12`.
- [ ] Change Realtime from `supabase/realtime:v2.51.11` to `supabase/realtime:v2.102.3`.
- [ ] Change Storage from `supabase/storage-api:v1.28.0` to `supabase/storage-api:v1.60.4`.
- [ ] Change imgproxy from `darthsim/imgproxy:v3.8.0` to `darthsim/imgproxy:v3.30.1`.
- [ ] Change postgres-meta from `supabase/postgres-meta:v0.91.6` to `supabase/postgres-meta:v0.96.6`.
- [ ] Rebuild the Railway-specific Postgres image from the `6ixfalls/supabase` `postgres` folder using `supabase/postgres:17.6.1.136` as the base image.
- [ ] For existing deployments, require a backup/restore or tested Postgres 15-to-17 upgrade plan before changing the Postgres image.

## 2. Add services missing from the template

- [ ] Add an Edge Functions service using `supabase/edge-runtime:v1.74.0`.
- [ ] Uncomment or add the existing `/functions/v1/*` Kong route and point it to the new Edge Functions service.
- [ ] Add a Supavisor service using the existing Railway-specific `pooler` source folder, rebased to `supabase/supavisor:2.9.5`.
- [ ] Add the Supavisor transaction-pooler TCP endpoint on port `6543` if Railway TCP proxies are desired.
- [ ] Update the existing `pooler.exs` rather than creating a second pooler config asset.
- [ ] Decide whether logs/analytics should be included now or delivered later as an optional add-on.

## 3. Refresh the existing Railway Postgres image

- [ ] Keep the `6ixfalls/supabase` Postgres image pattern because it already bakes the required SQL files into `/docker-entrypoint-initdb.d`.
- [ ] Refresh `_supabase.sql` from the official Docker setup.
- [ ] Refresh `logs.sql` from the official Docker setup.
- [ ] Refresh `pooler.sql` from the official Docker setup.
- [ ] Refresh `realtime.sql` from the official Docker setup.
- [ ] Refresh `roles.sql` from the official Docker setup.
- [ ] Refresh `jwt.sql` from the official Docker setup.
- [ ] Refresh `webhooks.sql` from the official Docker setup.
- [ ] Keep the Railway-specific `wrapper.sh` behavior for `PGHOST`, `PGPORT`, `PGDATA`, logging, and `/etc/postgresql-custom` persistence.
- [ ] Retest `wrapper.sh` against the Postgres 17 base image because the current wrapper references Postgres image internals from older upstream builds.

## 4. Replace unsafe or incomplete credential defaults

- [ ] Replace the blank Studio `AUTH_JWT_SECRET` default with a generated `JWT_SECRET` reference.
- [ ] Replace the blank Studio `SUPABASE_ANON_KEY` default with a generated `ANON_KEY` reference.
- [ ] Replace the blank Studio `SUPABASE_SERVICE_KEY` default with a generated `SERVICE_ROLE_KEY` reference.
- [ ] Replace hard-coded MinIO credentials with generated `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD` values.
- [ ] Keep `PG_META_CRYPTO_KEY` generated, but ensure the generated value is at least 32 characters.
- [ ] Ensure Realtime `DB_ENC_KEY` is exactly 16 characters.
- [ ] Ensure `SECRET_KEY_BASE` is at least 64 characters.
- [ ] Add `VAULT_ENC_KEY` for Supavisor and generate exactly 32 characters.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_ID` and `S3_PROTOCOL_ACCESS_KEY_SECRET` for the Storage S3 protocol endpoint.
- [ ] Mark `SERVICE_ROLE_KEY`, `SUPABASE_SECRET_KEY`, database passwords, SMTP passwords, S3 secrets, and pooler secrets as private/server-only in variable descriptions.

## 5. Add modern auth key support without removing legacy keys

- [ ] Add `SUPABASE_PUBLISHABLE_KEY` to Studio, Kong, and Edge Functions.
- [ ] Add `SUPABASE_SECRET_KEY` to Studio, Kong, and Edge Functions.
- [ ] Add optional `JWT_KEYS` support for Auth asymmetric signing.
- [ ] Add optional `JWT_JWKS` support for PostgREST JWT verification.
- [ ] Add optional `JWT_JWKS` support for Realtime JWT verification.
- [ ] Add optional `JWT_JWKS` support for Storage JWT verification.
- [ ] Add `ANON_KEY_ASYMMETRIC` to Kong.
- [ ] Add `SERVICE_ROLE_KEY_ASYMMETRIC` to Kong.
- [ ] Keep existing legacy `JWT_SECRET`, `ANON_KEY`, and `SERVICE_ROLE_KEY` compatibility for current clients.

## 6. Update Kong configuration

- [ ] Change `KONG_DNS_ORDER` from `AAAA,LAST,A,CNAME` to `LAST,A,CNAME` unless Railway requires IPv6-first DNS resolution.
- [ ] Add `KONG_DNS_NOT_FOUND_TTL=1`.
- [ ] Add `request-termination`, `ip-restriction`, and `post-function` to `KONG_PLUGINS`.
- [ ] Add `KONG_PROXY_ACCESS_LOG=/dev/stdout combined`.
- [ ] If keeping the existing Railway `envsubst` entrypoint, keep `KONG_DECLARATIVE_CONFIG=/home/kong/kong.yml`; only switch to `/usr/local/kong/kong.yml` if adopting the official Kong entrypoint layout.
- [ ] Keep the existing public service domain on Kong port `8000`.

## 7. Update Studio configuration

- [ ] Add `HOSTNAME=0.0.0.0`.
- [ ] Add `POSTGRES_PORT`.
- [ ] Add `POSTGRES_USER_READ_WRITE=postgres`.
- [ ] Add `PGRST_DB_SCHEMAS`.
- [ ] Add `PGRST_DB_MAX_ROWS`.
- [ ] Add `PGRST_DB_EXTRA_SEARCH_PATH`.
- [ ] Add optional `OPENAI_API_KEY`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEY`.
- [ ] Add `SUPABASE_SECRET_KEY`.
- [ ] Add `ENABLED_FEATURES_LOGS_ALL=false` unless logs are included and ready.
- [ ] Add snippets and Edge Functions management folders only if Railway can persist or mount those directories.

## 8. Update Auth configuration

- [ ] Add `GOTRUE_URI_ALLOW_LIST` from a new `ADDITIONAL_REDIRECT_URLS` variable.
- [ ] Add `GOTRUE_DISABLE_SIGNUP` from a new `DISABLE_SIGNUP` variable.
- [ ] Add `GOTRUE_JWT_EXP` from a new `JWT_EXPIRY` variable.
- [ ] Add `GOTRUE_JWT_ISSUER=${API_EXTERNAL_URL}/auth/v1`.
- [ ] Add optional `GOTRUE_JWT_KEYS` support.
- [ ] Add email signup and anonymous-user toggles.
- [ ] Add SMTP variables.
- [ ] Add mailer URL path variables.
- [ ] Add phone signup and phone autoconfirm toggles.
- [ ] Add optional OAuth provider variables.
- [ ] Add optional SMS provider variables.
- [ ] Add optional MFA variables.
- [ ] Add optional SAML variables.
- [ ] Add optional Auth hook variables.

## 9. Update PostgREST configuration

- [ ] Add `PGRST_DB_MAX_ROWS`.
- [ ] Add `PGRST_DB_EXTRA_SEARCH_PATH`.
- [ ] Add `PGRST_ADMIN_SERVER_PORT=3001`.
- [ ] Add `PGRST_ADMIN_SERVER_HOST=localhost`.
- [ ] Change `PGRST_JWT_SECRET` to use `JWT_JWKS` when present and `JWT_SECRET` otherwise.
- [ ] Change `PGRST_APP_SETTINGS_JWT_EXP` to use the shared `JWT_EXPIRY` variable.
- [ ] Remove `PGRST_SERVER_HOST=!6` unless a Railway-specific test proves it is required.

## 10. Update Realtime configuration

- [ ] Change `ERL_AFLAGS` from `-proto_dist inet6_tcp` to `-proto_dist inet_tcp` unless Railway requires IPv6 Erlang distribution.
- [ ] Rename or map `DB_ENC_KEY` to a shared `REALTIME_DB_ENC_KEY` variable.
- [ ] Add optional `API_JWT_JWKS` support.
- [ ] Add `METRICS_JWT_SECRET` from `JWT_SECRET`.
- [ ] Add `RUN_JANITOR=true`.
- [ ] Add `DISABLE_HEALTHCHECK_LOGGING=true`.
- [ ] Keep the existing `DB_AFTER_CONNECT_QUERY=SET search_path TO _realtime` setting.

## 11. Update Storage configuration

- [ ] Add `POSTGREST_URL` pointing to the private PostgREST URL on port `3000`.
- [ ] Add optional `JWT_JWKS` support.
- [ ] Uncomment or restore the Storage route in `kong/kong.yml` if Storage should be served through Kong; the current `6ixfalls` Kong config has it commented out.
- [ ] Add `STORAGE_PUBLIC_URL` from `SUPABASE_PUBLIC_URL`.
- [ ] Add `REQUEST_ALLOW_X_FORWARDED_PATH=true`.
- [ ] Replace `UPLOAD_FILE_SIZE_LIMIT` with `FILE_SIZE_LIMIT` unless Storage still accepts both in the target version.
- [ ] Replace `STORAGE_S3_BUCKET` with `GLOBAL_S3_BUCKET`.
- [ ] Add `FILE_STORAGE_BACKEND_PATH=/var/lib/storage` if using the official file backend.
- [ ] Replace `STORAGE_S3_REGION` with `REGION`.
- [ ] Replace `IMAGE_TRANSFORMATION_ENABLED` with `ENABLE_IMAGE_TRANSFORMATION` unless Storage still accepts both in the target version.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_ID`.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_SECRET`.
- [ ] If MinIO remains the default backend, map the old S3 variables to `GLOBAL_S3_ENDPOINT`, `GLOBAL_S3_PROTOCOL`, `GLOBAL_S3_FORCE_PATH_STYLE`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY`.
- [ ] If switching to the official file backend, add a persistent `/var/lib/storage` volume and set `STORAGE_BACKEND=file`.

## 12. Update imgproxy configuration

- [ ] Add `IMGPROXY_LOCAL_FILESYSTEM_ROOT=/`.
- [ ] Replace `IMGPROXY_ENABLE_WEBP_DETECTION=true` with `IMGPROXY_AUTO_WEBP=true` unless imgproxy still needs both.
- [ ] Add `IMGPROXY_MAX_SRC_RESOLUTION=16.8`.
- [ ] If Storage uses the file backend, mount the same storage volume into imgproxy.

## 13. Update postgres-meta configuration

- [ ] Change `PG_META_DB_USER` from `supabase_admin` to `postgres` if following the official Docker setup exactly.
- [ ] Keep the existing `CRYPTO_KEY`, host, port, database, and password wiring.

## 14. Configure Supavisor

- [ ] Add `DATABASE_URL=ecto://supabase_admin:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/_supabase`.
- [ ] Add `CLUSTER_POSTGRES=true`.
- [ ] Add `SECRET_KEY_BASE`.
- [ ] Add `VAULT_ENC_KEY`.
- [ ] Add `API_JWT_SECRET` from `JWT_SECRET`.
- [ ] Add `METRICS_JWT_SECRET` from `JWT_SECRET`.
- [ ] Add `REGION=local` unless Railway-specific regions are required.
- [ ] Add `ERL_AFLAGS=-proto_dist inet_tcp` unless Railway requires IPv6 Erlang distribution.
- [ ] Add `POOLER_TENANT_ID`.
- [ ] Add `POOLER_DEFAULT_POOL_SIZE=20`.
- [ ] Add `POOLER_MAX_CLIENT_CONN=100`.
- [ ] Add `POOLER_POOL_MODE=transaction`.
- [ ] Add `DB_POOL_SIZE=5`.
- [ ] Keep the existing Railway pooler startup command, but retest it after rebasing from `supabase/supavisor:2.7.0` to `supabase/supavisor:2.9.5`.

## 15. Configure Edge Functions

- [ ] Add `SUPABASE_PUBLIC_URL`.
- [ ] Add `SUPABASE_SERVICE_ROLE_KEY` from `SERVICE_ROLE_KEY`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEYS` using `SUPABASE_PUBLISHABLE_KEY`.
- [ ] Add `SUPABASE_SECRET_KEYS` using `SUPABASE_SECRET_KEY`.
- [ ] Add `SUPABASE_DB_URL=postgresql://postgres:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}`.
- [ ] Add `VERIFY_JWT` from `FUNCTIONS_VERIFY_JWT`.
- [ ] Provide `/home/deno/functions/main`.
- [ ] Start Edge Runtime with `start --main-service /home/deno/functions/main`.

## 16. Add missing health checks

- [ ] Add a Studio health check that verifies `/api/platform/profile` returns `200`.
- [ ] Add a Kong health check using `kong health` or an equivalent route.
- [ ] Add an Auth health check against `/health` on port `9999`.
- [ ] Add a PostgREST readiness check using `postgrest --ready` or the admin server.
- [ ] Add a Realtime tenant health check that sends an anon bearer token.
- [ ] Add an imgproxy health check using `imgproxy health`.
- [ ] Add a Postgres health check using `pg_isready`.
- [ ] Add a Supavisor health check against `/api/health`.
- [ ] Add an Edge Functions TCP health check on port `9000`.
- [ ] Keep the existing Storage `/status` health check.
- [ ] Keep the existing MinIO `/minio/health/ready` health check if MinIO remains in the template.

## 17. Validate only changed behavior

- [ ] Deploy a fresh project from the updated template.
- [ ] Confirm the newly added Edge Functions service becomes healthy.
- [ ] Confirm the newly added Supavisor service becomes healthy.
- [ ] Confirm the updated Kong image routes existing `/auth/v1/*`, `/rest/v1/*`, `/realtime/v1/*`, and `/storage/v1/*` paths.
- [ ] Confirm Kong routes the new `/functions/v1/*` path.
- [ ] Confirm generated `ANON_KEY` can query PostgREST.
- [ ] Confirm generated `SERVICE_ROLE_KEY` works server-side and is not exposed in browser-visible variables.
- [ ] Confirm Realtime works after the version and environment updates.
- [ ] Confirm Storage works after the variable rename/mapping updates.
- [ ] Confirm Supavisor accepts session-mode and transaction-mode Postgres connections if both are exposed.
- [ ] Document any Railway-specific deviations from the official Docker setup.
