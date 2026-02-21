# Branch: `feature/immich-db-from-nextcloud` — Immich DB from Postgres 1Password item

## Summary
Use the same 1Password item as the Postgres cluster (**postgresql** / **password**) for Immich DB credentials. No separate 1Password item for Immich.

## Changes
- **ExternalSecret** for `immich-db-credentials` from 1Password item **postgresql** (property **password**). Secret has `DB_USERNAME=postgres` and `DB_PASSWORD`.
- **onepassworditem** for DB disabled: `onepassworditem.secrets.immich: []`.
- **README** updated: single source of truth, create database `immich` only; ESO required.

## 1Password
- No new item. Use existing **postgresql** item (same as Postgres cluster and Harbor).

## Postgres
- Create database: `CREATE DATABASE immich;` (postgres user already exists).
