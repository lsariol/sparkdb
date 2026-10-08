# Changelog

All notable changes to sparkdb. Versions follow [semantic versioning](https://semver.org).

## v1.0.0

The first release deployed by Lighthouse v1.0.0.

### Added
- **Lighthouse settings:** `x-lighthouse: {deploy: releases, tier: data, backup: postgres}`. sparkdb deploys only when a version is tagged, before everything else and alone, after Lighthouse dumps every database to `/srv/backups/sparkdb/`.
- A healthcheck (`pg_isready`), so a deploy waits until Postgres accepts connections and rolls back if it doesn't.

### Changed
- **The data lives in `/srv/server/storage/sparkdb/data`**, the project's own storage folder, instead of `/srv/pg_data`.
- **The port is published to the server only** (`127.0.0.1:5000`); pgAdmin connects through an SSH tunnel.
- The admin password comes from Cove as `SPARKDB_DATABASE_PASSWORD`; the admin user (`Admin`) and default database (`Admin_DB`) are written in the compose file.

### Upgrading
- Move the data folder to `/srv/server/storage/sparkdb/data` with sparkdb stopped, before deploying.
- pgAdmin needs an SSH tunnel: host `127.0.0.1`, port `5000`.
