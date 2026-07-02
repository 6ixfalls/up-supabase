# Railway Supabase template update checklist

This checklist only includes template changes that are still needed after accounting for the existing Railway template and the Railway-specific `github.com/6ixfalls/supabase` source repository.

## Railway source repository findings

Keep the Railway-specific source assets and rebase them instead of recreating them:

- `kong/Dockerfile` already copies `kong.yml`, installs `gettext`, and renders Railway variables with `envsubst`.
- `kong/kong.yml` already has Auth, REST, GraphQL, Realtime, Analytics, postgres-meta, and Studio routes; Storage and Edge Functions routes need to be enabled/updated.
- `postgres/Dockerfile` already bakes Supabase init SQL into `/docker-entrypoint-initdb.d` and installs `wrapper.sh`.
- `postgres/wrapper.sh` already handles Railway `PGHOST`/`PGPORT` behavior and persists `/etc/postgresql-custom` through the Postgres data volume.
- `pooler/Dockerfile` already bakes `pooler.exs` into a Supavisor image and starts Supavisor with migrate, eval, and server commands.

## Credential generator

Use `docs/supabase-credential-generator.html` to generate the source credentials. Paste the generated values into **Supabase Studio** variables, then reference them from every other service with Railway references.

Source credentials generated under **Supabase Studio**:

- `JWT_SECRET`
- `ANON_KEY`
- `SERVICE_ROLE_KEY`
- `SECRET_KEY_BASE`
- `REALTIME_DB_ENC_KEY`
- `VAULT_ENC_KEY`
- `PG_META_CRYPTO_KEY`
- `S3_PROTOCOL_ACCESS_KEY_ID`
- `S3_PROTOCOL_ACCESS_KEY_SECRET`
- `MINIO_ROOT_USER`
- `MINIO_ROOT_PASSWORD`
- `POSTGRES_PASSWORD`
- `DASHBOARD_PASSWORD`
- `SUPABASE_PUBLISHABLE_KEY`
- `SUPABASE_SECRET_KEY`
- `ANON_KEY_ASYMMETRIC`
- `SERVICE_ROLE_KEY_ASYMMETRIC`
- `JWT_KEYS`
- `JWT_JWKS`

Do not create duplicate generated secrets on Postgres, Kong, Auth, Realtime, Storage, or Supavisor. Reference the Studio source variables instead.

## 1. Rebase images and Railway source folders

- [ ] Update `6ixfalls/supabase/kong/Dockerfile` from `kong:2.8.1` to `kong/kong:3.9.1`.
- [ ] Update Studio from `supabase/studio:2025.10.09-sha-433e578` to `supabase/studio:2026.06.03-sha-0bca601`.
- [ ] Update Auth from `supabase/gotrue:v2.180.0` to `supabase/gotrue:v2.189.0`.
- [ ] Update PostgREST from `postgrest/postgrest:v13.0.7` to `postgrest/postgrest:v14.12`.
- [ ] Update Realtime from `supabase/realtime:v2.51.11` to `supabase/realtime:v2.102.3`.
- [ ] Update Storage from `supabase/storage-api:v1.28.0` to `supabase/storage-api:v1.60.4`.
- [ ] Update imgproxy from `darthsim/imgproxy:v3.8.0` to `darthsim/imgproxy:v3.30.1`.
- [ ] Update postgres-meta from `supabase/postgres-meta:v0.91.6` to `supabase/postgres-meta:v0.96.6`.
- [ ] Rebuild the Railway Postgres image from `6ixfalls/supabase/postgres` using `supabase/postgres:17.6.1.136` as the base image.
- [ ] Refresh the SQL files already baked by `6ixfalls/supabase/postgres` from the official Docker setup: `_supabase.sql`, `logs.sql`, `pooler.sql`, `realtime.sql`, `roles.sql`, `jwt.sql`, and `webhooks.sql`.
- [ ] Rebase `6ixfalls/supabase/pooler` from `supabase/supavisor:2.7.0` to `supabase/supavisor:2.9.5`.

## 2. Add only missing services

- [ ] Add Edge Functions with `supabase/edge-runtime:v1.74.0`.
- [ ] Add Supavisor using the existing `6ixfalls/supabase/pooler` source folder.
- [ ] Add a Supavisor transaction TCP proxy on `6543` only if Railway users need transaction pooling.
- [ ] Leave logs/analytics out unless the template will expose Logflare/Vector intentionally.

## 3. Supabase Studio variables

Keep existing Studio UI variables, but add only the missing Docker-aligned values below.

- [ ] Add `ANON_KEY` from the credential generator.
- [ ] Add `SERVICE_ROLE_KEY` from the credential generator.
- [ ] Add `JWT_SECRET` from the credential generator.
- [ ] Add `POSTGRES_PASSWORD` from the credential generator.
- [ ] Add `SECRET_KEY_BASE` from the credential generator.
- [ ] Add `REALTIME_DB_ENC_KEY` from the credential generator.
- [ ] Add `VAULT_ENC_KEY` from the credential generator.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_ID` from the credential generator.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_SECRET` from the credential generator.
- [ ] Add `MINIO_ROOT_USER` from the credential generator.
- [ ] Add `MINIO_ROOT_PASSWORD` from the credential generator.
- [ ] Add `DASHBOARD_PASSWORD` from the credential generator.
- [ ] Add `SUPABASE_PUBLISHABLE_KEY` from the credential generator.
- [ ] Add `SUPABASE_SECRET_KEY` from the credential generator.
- [ ] Add `JWT_KEYS` from the credential generator.
- [ ] Add `JWT_JWKS` from the credential generator.
- [ ] Change Studio `AUTH_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Change Studio `SUPABASE_ANON_KEY` to `${{"Supabase Studio".ANON_KEY}}`.
- [ ] Change Studio `SUPABASE_SERVICE_KEY` to `${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Keep `SUPABASE_PUBLIC_URL=https://${{Kong.RAILWAY_PUBLIC_DOMAIN}}`.
- [ ] Add `POSTGRES_PORT=${{Postgres.PGPORT}}`.
- [ ] Add `PGRST_DB_SCHEMAS=${{Postgrest.PGRST_DB_SCHEMAS}}`.
- [ ] Add `PGRST_DB_MAX_ROWS=1000` only if Studio needs to edit this value.
- [ ] Add `PGRST_DB_EXTRA_SEARCH_PATH=public` only if Studio needs to edit this value.

## 4. Postgres variables

Use the generated Studio password rather than a separate Postgres secret.

- [ ] Change `POSTGRES_PASSWORD` to `${{"Supabase Studio".POSTGRES_PASSWORD}}`.
- [ ] Keep `PGPASSWORD=${{POSTGRES_PASSWORD}}`.
- [ ] Keep `PGUSER=${{POSTGRES_USER}}`.
- [ ] Keep `PGDATABASE=${{POSTGRES_DB}}`.
- [ ] Change `JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Keep `JWT_EXP=${{Postgrest.PGRST_APP_SETTINGS_JWT_EXP}}`.

## 5. Kong variables and routes

Keep Railway's `envsubst` Kong flow. Do not switch `KONG_DECLARATIVE_CONFIG` unless the source repo changes its rendered file path.

- [ ] Keep `KONG_DECLARATIVE_CONFIG=/home/kong/kong.yml` for the current `6ixfalls/supabase/kong` flow.
- [ ] Change `SUPABASE_ANON_KEY` to `${{"Supabase Studio".ANON_KEY}}`.
- [ ] Change `SUPABASE_SERVICE_KEY` to `${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEY=${{"Supabase Studio".SUPABASE_PUBLISHABLE_KEY}}`.
- [ ] Add `SUPABASE_SECRET_KEY=${{"Supabase Studio".SUPABASE_SECRET_KEY}}`.
- [ ] Add `ANON_KEY_ASYMMETRIC=${{"Supabase Studio".ANON_KEY_ASYMMETRIC}}`.
- [ ] Add `SERVICE_ROLE_KEY_ASYMMETRIC=${{"Supabase Studio".SERVICE_ROLE_KEY_ASYMMETRIC}}`.
- [ ] Change `DASHBOARD_PASSWORD` to `${{"Supabase Studio".DASHBOARD_PASSWORD}}`.
- [ ] Keep `DASHBOARD_USERNAME` generated or user-defined; do not add a second dashboard username variable elsewhere.
- [ ] Add `request-termination`, `ip-restriction`, and `post-function` to `KONG_PLUGINS`.
- [ ] Change `KONG_DNS_ORDER` to `LAST,A,CNAME` unless Railway IPv6 resolution requires the current value.
- [ ] Add `KONG_DNS_NOT_FOUND_TTL=1` only if supported by the Railway Kong image.
- [ ] Restore the Storage route in `kong/kong.yml` if Storage should be served through Kong.
- [ ] Add the Functions route in `kong/kong.yml` for `/functions/v1/*`.

## 6. Auth variables

Only add variables that are missing from the template and likely to be user-configured.

- [ ] Change `GOTRUE_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `GOTRUE_JWT_EXP=${{Postgrest.PGRST_APP_SETTINGS_JWT_EXP}}`.
- [ ] Add `GOTRUE_JWT_ISSUER=${{API_EXTERNAL_URL}}/auth/v1`.
- [ ] Add `GOTRUE_URI_ALLOW_LIST` for additional redirect URLs.
- [ ] Add `GOTRUE_DISABLE_SIGNUP` only if the template should expose signup control.
- [ ] Add `GOTRUE_JWT_KEYS=${{"Supabase Studio".JWT_KEYS}}` only when asymmetric auth is enabled.
- [ ] Add SMTP variables only if the template will support email delivery out of the box.
- [ ] Add OAuth/SMS/MFA/SAML/hook variables only as optional examples, not required template variables.

## 7. PostgREST variables

- [ ] Change `PGRST_JWT_SECRET` to `${{"Supabase Studio".JWT_JWKS}}` when asymmetric auth is enabled; otherwise keep `${{"Supabase Studio".JWT_SECRET}}` style legacy wiring.
- [ ] Change `PGRST_APP_SETTINGS_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Keep `PGRST_APP_SETTINGS_JWT_EXP=3600` unless the template exposes JWT expiry as a user setting.
- [ ] Add `PGRST_DB_MAX_ROWS=1000` only if users need to edit it.
- [ ] Add `PGRST_DB_EXTRA_SEARCH_PATH=public` only if users need to edit it.
- [ ] Remove `PGRST_SERVER_HOST=!6` unless Railway proves it is required for PostgREST v14.

## 8. Realtime variables

- [ ] Change `DB_ENC_KEY` to `${{"Supabase Studio".REALTIME_DB_ENC_KEY}}`.
- [ ] Change `API_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `API_JWT_JWKS=${{"Supabase Studio".JWT_JWKS}}` only when asymmetric auth is enabled.
- [ ] Add `METRICS_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `RUN_JANITOR=true`.
- [ ] Add `DISABLE_HEALTHCHECK_LOGGING=true`.
- [ ] Change `ERL_AFLAGS` to `-proto_dist inet_tcp` unless Railway requires IPv6 Erlang distribution.

## 9. Storage variables

Keep S3/MinIO because the Railway template already uses it. Do not add file-backend variables unless switching away from S3.

- [ ] Change `ANON_KEY` to `${{"Supabase Studio".ANON_KEY}}`.
- [ ] Change `SERVICE_KEY` to `${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Change `AUTH_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `JWT_JWKS=${{"Supabase Studio".JWT_JWKS}}` only when asymmetric auth is enabled.
- [ ] Add `POSTGREST_URL=http://${{Postgrest.RAILWAY_PRIVATE_DOMAIN}}:3000`.
- [ ] Add `STORAGE_PUBLIC_URL=${{"Supabase Studio".SUPABASE_PUBLIC_URL}}`.
- [ ] Add `REQUEST_ALLOW_X_FORWARDED_PATH=true`.
- [ ] Rename `UPLOAD_FILE_SIZE_LIMIT` to `FILE_SIZE_LIMIT` if required by `storage-api:v1.60.4`.
- [ ] Rename `STORAGE_S3_BUCKET` to `GLOBAL_S3_BUCKET` if required by `storage-api:v1.60.4`.
- [ ] Rename `STORAGE_S3_REGION` to `REGION` if required by `storage-api:v1.60.4`.
- [ ] Rename `IMAGE_TRANSFORMATION_ENABLED` to `ENABLE_IMAGE_TRANSFORMATION` if required by `storage-api:v1.60.4`.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_ID=${{"Supabase Studio".S3_PROTOCOL_ACCESS_KEY_ID}}`.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_SECRET=${{"Supabase Studio".S3_PROTOCOL_ACCESS_KEY_SECRET}}`.

## 10. S3 / MinIO variables

- [ ] Change `MINIO_ROOT_USER` to `${{"Supabase Studio".MINIO_ROOT_USER}}`.
- [ ] Change `MINIO_ROOT_PASSWORD` to `${{"Supabase Studio".MINIO_ROOT_PASSWORD}}`.
- [ ] Keep existing Railway private endpoint variables; they are service-derived and should not be duplicated in Studio.

## 11. imgproxy and postgres-meta variables

- [ ] For imgproxy, add only `IMGPROXY_LOCAL_FILESYSTEM_ROOT=/` and `IMGPROXY_MAX_SRC_RESOLUTION=16.8` if required by the newer image.
- [ ] For imgproxy, rename `IMGPROXY_ENABLE_WEBP_DETECTION` to `IMGPROXY_AUTO_WEBP` if required by `darthsim/imgproxy:v3.30.1`.
- [ ] For postgres-meta, change `CRYPTO_KEY` to `${{"Supabase Studio".PG_META_CRYPTO_KEY}}`.
- [ ] For postgres-meta, keep existing Postgres host, port, database, user, and password references unless the official role change is required.

## 12. Supavisor variables

Add Supavisor as a new service; source secrets from Studio.

- [ ] Add `DATABASE_URL=ecto://supabase_admin:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/_supabase`.
- [ ] Add `SECRET_KEY_BASE=${{"Supabase Studio".SECRET_KEY_BASE}}`.
- [ ] Add `VAULT_ENC_KEY=${{"Supabase Studio".VAULT_ENC_KEY}}`.
- [ ] Add `API_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `METRICS_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `POOLER_TENANT_ID` as a generated or user-provided non-secret value.
- [ ] Add pool sizes only if the template should make them user configurable; otherwise keep them in `pooler.exs`.

## 13. Edge Functions variables

Add Edge Functions as a new service; source secrets from Studio.

- [ ] Add `JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `SUPABASE_URL=http://${{Kong.RAILWAY_PRIVATE_DOMAIN}}:8000`.
- [ ] Add `SUPABASE_PUBLIC_URL=${{"Supabase Studio".SUPABASE_PUBLIC_URL}}`.
- [ ] Add `SUPABASE_ANON_KEY=${{"Supabase Studio".ANON_KEY}}`.
- [ ] Add `SUPABASE_SERVICE_ROLE_KEY=${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEYS={"default":"${{"Supabase Studio".SUPABASE_PUBLISHABLE_KEY}}"}`.
- [ ] Add `SUPABASE_SECRET_KEYS={"default":"${{"Supabase Studio".SUPABASE_SECRET_KEY}}"}`.
- [ ] Add `SUPABASE_DB_URL=postgresql://postgres:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/${{Postgres.PGDATABASE}}`.
- [ ] Add `VERIFY_JWT=false` unless the template should expose this as a user setting.

## 14. Health checks and validation

- [ ] Add missing health checks for Studio, Kong, Auth, PostgREST, Realtime, imgproxy, Postgres, Supavisor, and Edge Functions.
- [ ] Keep existing Storage and MinIO health checks.
- [ ] Validate a fresh deployment by checking Studio, Auth, REST, Realtime, Storage, Edge Functions, and Supavisor transaction pooling.
- [ ] Validate that browser-visible clients only receive anon/publishable keys and never receive service-role or secret keys.
