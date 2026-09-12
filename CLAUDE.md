# CLAUDE.md - kyte-postgres

## What this is

A PostgreSQL wire-protocol driver for the Kyte language, shipped as a Kyte **package** (not part of the
compiler repo). It speaks the v3 frontend/backend protocol on Kyte's async runtime: SCRAM-SHA-256 (and
cleartext) auth, server-side prepared statements, TLS, and a streaming cursor. It implements the stdlib
`Driver` / `Connection` seam, so an application or the ORM talks to it through `db`, never through its
internals. Module/import name: `postgres` (`import postgres;`). See [README.md](README.md) for the
overview.

## Build and test

The package is compiled by the installed Kyte toolchain (`~/.kyte/bin/kyte`, built ReleaseFast from the
`kyte` repo); there is no separate build step for the package itself. Run the tests from the repo root:

```sh
~/.kyte/bin/kyte test                 # offline suite: codec / SCRAM / prepared-statement unit tests
```

The live gate (connect + query + close loop against a real server) is gated on `KYTE_PG_URL` and skips
when it is unset, so a plain `kyte test` stays green with no database:

```sh
KYTE_PG_URL='postgresql://postgres:postgres@127.0.0.1:5432/postgres?sslmode=disable' ~/.kyte/bin/kyte test
```

`tests/66_postgres_codec.ky` is the offline codec gate (hand-built byte buffers); `107`/`112` cover SCRAM,
`108` LISTEN/COPY, `113` reader-buffer free, `114` a live TLS connection.

## Working in this repo (how to make a change)

1. **Understand, then plan.** Read the relevant `src/` module and the test that exercises it before
   editing. Keep the change minimal and match the surrounding Kyte style; do not reformat unrelated code.
2. **Verify real behaviour, not just compilation.** "It compiles" is not done. Exercise the offline codec
   path with `kyte test`, and for anything touching the wire or auth, run the live gate with `KYTE_PG_URL`
   pointed at a real PostgreSQL (16 is what CI uses).
3. **Run the test suite before and after.** `~/.kyte/bin/kyte test` at minimum; the live loop is the gate
   for connection, auth, and codec changes. Add a test for every behaviour change (a codec change gets an
   offline byte-exact case; a wire change gets a live case).
4. **Commit only when asked**, and branch first if you are on `main`. The compiler, the other drivers, and
   kynator are separate kytelang repos: a change to any of those does not belong here.

## Layout map

- `src/postgres.ky` - the driver interface: `PgConnection impl Connection`, `PgDriver impl Driver`, and
  the connect/handshake orchestration (the only surface a consumer touches).
- `src/connection.ky` - connection-string parsing (`ConnectionOptions`, `parse`), including percent-decode
  of userinfo and `sslmode` / `sslrootcert`.
- `src/codec.ky` - the v3 wire codec: frame builders, response decoders, `PgCursor`.
- `src/typemap.ky` - OID to `DbType`, wire-text to `DbValue`, `DbValue` to SQL/bind text, `substituteParams`.
- `src/proto.ky` - async transport framing (`PgReader`, `PgFrame`, `readFrame`, `sendFrame`).
- `src/stmt.ky` - prepared-statement cache entry (`PgStmt`). `src/auth.ky` - SCRAM-SHA-256 / cleartext.
- `tests/` - Kyte test cases (offline codec gates + live `KYTE_PG_URL` gates).

## Conventions and gotchas

- The pure halves (`codec` / `typemap`) are offline-gated; `connect` / `query` are only meaningful against
  a running server, so validate protocol changes live, not only through the offline cases.
- Prose follows Indian English with British spellings (behaviour, colour, initialise) and no em dashes;
  never change code identifiers or API names to match.

## Ecosystem context

A Kyte package consumed as a **git-URL dependency** (no central registry). A consumer fetches it with
`kyte get https://github.com/kytelang/kyte-postgres`, which records it in the manifest `project.json` and
pins it in the lockfile `project.lock.json`; the import name is then `postgres`. The compiler that builds
it lives in the `kyte` repo; sibling packages (kyte-mysql, kyte-mssql, kyte-mongodb, kyte-datastar) are
separate repos.
