# Railway Supabase template upgrade checklist

This checklist contains only actions needed to upgrade the supplied Railway Supabase template while preserving the existing Railway-specific `github.com/6ixfalls/supabase` source assets.

## Credential generator

Use `docs/supabase-credential-generator.html` only for values Railway cannot derive with `random(len, "charset")`: JWT-derived API keys, JWKS values, and opaque Supabase API keys. Paste these generated values into **Supabase Studio** variables and reference them from other services with Railway references.

Generate under **Supabase Studio** with the HTML tool:

- `JWT_SECRET`
- `ANON_KEY`
- `SERVICE_ROLE_KEY`
- `SUPABASE_PUBLISHABLE_KEY`
- `SUPABASE_SECRET_KEY`
- `ANON_KEY_ASYMMETRIC`
- `SERVICE_ROLE_KEY_ASYMMETRIC`
- `JWT_KEYS`
- `JWT_JWKS`

Use Railway random expressions directly on the service variable that consumes the independent secret:

- Passwords: `${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789") }}`
- 16-character hex keys: `${{ random(16, "0123456789abcdef") }}`
- 32-character hex keys: `${{ random(32, "0123456789abcdef") }}`
- 64-character base64url-like keys: `${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-_") }}`

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
- [ ] Refresh the SQL files baked by `6ixfalls/supabase/postgres`: `_supabase.sql`, `logs.sql`, `pooler.sql`, `realtime.sql`, `roles.sql`, `jwt.sql`, and `webhooks.sql`.
- [ ] Rebase `6ixfalls/supabase/pooler` from `supabase/supavisor:2.7.0` to `supabase/supavisor:2.9.5`.

## 2. Add missing services

- [ ] Add Edge Functions with `supabase/edge-runtime:v1.74.0`.
- [ ] Add Supavisor using the existing `6ixfalls/supabase/pooler` source folder.
- [ ] Add a Supavisor transaction TCP proxy on `6543` if Railway users need transaction pooling.

## 3. Supabase Studio variables

- [ ] Add `ANON_KEY` from the credential generator.
- [ ] Add `SERVICE_ROLE_KEY` from the credential generator.
- [ ] Add `JWT_SECRET` from the credential generator.
- [ ] Add `SUPABASE_PUBLISHABLE_KEY` from the credential generator.
- [ ] Add `SUPABASE_SECRET_KEY` from the credential generator.
- [ ] Add `JWT_KEYS` from the credential generator.
- [ ] Add `JWT_JWKS` from the credential generator.
- [ ] Change Studio `AUTH_JWT_SECRET` to `${{JWT_SECRET}}`.
- [ ] Change Studio `SUPABASE_ANON_KEY` to `${{ANON_KEY}}`.
- [ ] Change Studio `SUPABASE_SERVICE_KEY` to `${{SERVICE_ROLE_KEY}}`.
- [ ] Add `POSTGRES_PORT=${{Postgres.PGPORT}}`.
- [ ] Add `PGRST_DB_SCHEMAS=${{Postgrest.PGRST_DB_SCHEMAS}}`.

## 4. Postgres variables

- [ ] Change `POSTGRES_PASSWORD` to `${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789") }}`.
- [ ] Change `JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.

## 5. Kong variables and routes

- [ ] Change `SUPABASE_ANON_KEY` to `${{"Supabase Studio".ANON_KEY}}`.
- [ ] Change `SUPABASE_SERVICE_KEY` to `${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEY=${{"Supabase Studio".SUPABASE_PUBLISHABLE_KEY}}`.
- [ ] Add `SUPABASE_SECRET_KEY=${{"Supabase Studio".SUPABASE_SECRET_KEY}}`.
- [ ] Add `ANON_KEY_ASYMMETRIC=${{"Supabase Studio".ANON_KEY_ASYMMETRIC}}`.
- [ ] Add `SERVICE_ROLE_KEY_ASYMMETRIC=${{"Supabase Studio".SERVICE_ROLE_KEY_ASYMMETRIC}}`.
- [ ] Change `DASHBOARD_PASSWORD` to `${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789") }}`.
- [ ] Add `request-termination`, `ip-restriction`, and `post-function` to `KONG_PLUGINS`.
- [ ] Change `KONG_DNS_ORDER` to `LAST,A,CNAME` unless Railway IPv6 resolution requires the current value.
- [ ] Add `KONG_DNS_NOT_FOUND_TTL=1` if supported by the Railway Kong image.
- [ ] Restore the Storage route in `kong/kong.yml`.
- [ ] Add the Functions route in `kong/kong.yml` for `/functions/v1/*`.

## 6. Auth variables

- [ ] Change `GOTRUE_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `GOTRUE_JWT_EXP=${{Postgrest.PGRST_APP_SETTINGS_JWT_EXP}}`.
- [ ] Add `GOTRUE_JWT_ISSUER=${{API_EXTERNAL_URL}}/auth/v1`.
- [ ] Add `GOTRUE_URI_ALLOW_LIST` for additional redirect URLs.
- [ ] Add `GOTRUE_JWT_KEYS=${{"Supabase Studio".JWT_KEYS}}` when asymmetric auth is enabled.

## 7. PostgREST variables

- [ ] Change `PGRST_JWT_SECRET` to `${{"Supabase Studio".JWT_JWKS}}` when asymmetric auth is enabled.
- [ ] Change `PGRST_APP_SETTINGS_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Remove `PGRST_SERVER_HOST=!6` unless Railway proves it is required for PostgREST v14.

## 8. Realtime variables

- [ ] Change `DB_ENC_KEY` to `${{ random(16, "0123456789abcdef") }}`.
- [ ] Change `API_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `API_JWT_JWKS=${{"Supabase Studio".JWT_JWKS}}` when asymmetric auth is enabled.
- [ ] Add `METRICS_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `RUN_JANITOR=true`.
- [ ] Add `DISABLE_HEALTHCHECK_LOGGING=true`.
- [ ] Change `ERL_AFLAGS` to `-proto_dist inet_tcp` unless Railway requires IPv6 Erlang distribution.

## 9. Storage variables

- [ ] Change `ANON_KEY` to `${{"Supabase Studio".ANON_KEY}}`.
- [ ] Change `SERVICE_KEY` to `${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Change `AUTH_JWT_SECRET` to `${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `JWT_JWKS=${{"Supabase Studio".JWT_JWKS}}` when asymmetric auth is enabled.
- [ ] Add `POSTGREST_URL=http://${{Postgrest.RAILWAY_PRIVATE_DOMAIN}}:3000`.
- [ ] Add `STORAGE_PUBLIC_URL=${{"Supabase Studio".SUPABASE_PUBLIC_URL}}`.
- [ ] Add `REQUEST_ALLOW_X_FORWARDED_PATH=true`.
- [ ] Rename `UPLOAD_FILE_SIZE_LIMIT` to `FILE_SIZE_LIMIT` if required by `storage-api:v1.60.4`.
- [ ] Rename `STORAGE_S3_BUCKET` to `GLOBAL_S3_BUCKET` if required by `storage-api:v1.60.4`.
- [ ] Rename `STORAGE_S3_REGION` to `REGION` if required by `storage-api:v1.60.4`.
- [ ] Rename `IMAGE_TRANSFORMATION_ENABLED` to `ENABLE_IMAGE_TRANSFORMATION` if required by `storage-api:v1.60.4`.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_ID=${{ random(32, "0123456789abcdef") }}`.
- [ ] Add `S3_PROTOCOL_ACCESS_KEY_SECRET=${{ random(64, "0123456789abcdef") }}`.

## 10. S3 / MinIO variables

- [ ] Change `MINIO_ROOT_USER` to `${{ random(32, "0123456789abcdef") }}`.
- [ ] Change `MINIO_ROOT_PASSWORD` to `${{ random(64, "0123456789abcdef") }}`.

## 11. imgproxy and postgres-meta variables

- [ ] Add `IMGPROXY_LOCAL_FILESYSTEM_ROOT=/` if required by `darthsim/imgproxy:v3.30.1`.
- [ ] Add `IMGPROXY_MAX_SRC_RESOLUTION=16.8` if required by `darthsim/imgproxy:v3.30.1`.
- [ ] Rename `IMGPROXY_ENABLE_WEBP_DETECTION` to `IMGPROXY_AUTO_WEBP` if required by `darthsim/imgproxy:v3.30.1`.
- [ ] Change postgres-meta `CRYPTO_KEY` to `${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-_") }}`.

## 12. Supavisor variables

- [ ] Add `DATABASE_URL=ecto://supabase_admin:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/_supabase`.
- [ ] Add `SECRET_KEY_BASE=${{ random(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-_") }}`.
- [ ] Add `VAULT_ENC_KEY=${{ random(32, "0123456789abcdef") }}`.
- [ ] Add `API_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `METRICS_JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `POOLER_TENANT_ID` as a generated or user-provided non-secret value.

## 13. Edge Functions variables

- [ ] Add `JWT_SECRET=${{"Supabase Studio".JWT_SECRET}}`.
- [ ] Add `SUPABASE_URL=http://${{Kong.RAILWAY_PRIVATE_DOMAIN}}:8000`.
- [ ] Add `SUPABASE_PUBLIC_URL=${{"Supabase Studio".SUPABASE_PUBLIC_URL}}`.
- [ ] Add `SUPABASE_ANON_KEY=${{"Supabase Studio".ANON_KEY}}`.
- [ ] Add `SUPABASE_SERVICE_ROLE_KEY=${{"Supabase Studio".SERVICE_ROLE_KEY}}`.
- [ ] Add `SUPABASE_PUBLISHABLE_KEYS={"default":"${{"Supabase Studio".SUPABASE_PUBLISHABLE_KEY}}"}`.
- [ ] Add `SUPABASE_SECRET_KEYS={"default":"${{"Supabase Studio".SUPABASE_SECRET_KEY}}"}`.
- [ ] Add `SUPABASE_DB_URL=postgresql://postgres:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/${{Postgres.PGDATABASE}}`.

## 14. Health checks and validation

- [ ] Add missing health checks for Studio, Kong, Auth, PostgREST, Realtime, imgproxy, Postgres, Supavisor, and Edge Functions.
- [ ] Validate a fresh deployment by checking Studio, Auth, REST, Realtime, Storage, Edge Functions, and Supavisor transaction pooling.
- [ ] Validate that browser-visible clients only receive anon/publishable keys and never receive service-role or secret keys.
