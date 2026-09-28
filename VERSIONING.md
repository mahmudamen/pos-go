# Versioning

How POS.Go versions its releases, and what a merchant can expect.

## Two version lines

| Line | Example | Meaning | Who sees it |
|---|---|---|---|
| **App version** | `1.0.0` | The Android build. Requires an update from Google Play. | Everyone |
| **`versionCode`** | `3` | A monotonically increasing integer Google Play uses to order uploads. Never shown to users. | Google Play only |
| **Platform version** | `v1.2` | The hosted backend and admin console. | Nobody — it just works |

The platform and the app are deployed independently. A platform release is
transparent to a merchant unless its notes say otherwise; an app release may
require a Play update, and Play can also stage an update for days before it
reaches a device.

## App versions

Semantic versioning, `MAJOR.MINOR.PATCH`:

- **PATCH** — bug fixes only. No new features, no behaviour changes, no schema
  changes on the server.
- **MINOR** — new features, backwards compatible. Existing stores keep working
  without configuration. A minor release may add a new server capability, but
  must not require it.
- **MAJOR** — a change a store would notice as breaking. Reserved for changes that
  force a merchant to act: a required migration, a removed field, a changed
  meaning, or a new mandatory step.

Every build bumps `versionCode`, including releases that change nothing visible.
It is the ordering key, so it must increase on every single upload.

Patch and minor releases are additive. If a release needs a merchant to change
something, it is a major release.

## `versionCode` policy

- Strictly increasing. Never reused, never reset, never rolled back.
- The currently published artifact is `1.0.0`, `versionCode 3`.
- The first build against an already-published `versionCode` **must** increase it,
  even if the version string is unchanged. A lower or equal `versionCode` is
  rejected by Play.
- Internal development builds and the universal APK used for manual sideloading
  do not consume a `versionCode`. Only Play-bound artifacts do.

## What a merchant can rely on

- **Data is never lost across a version bump.** Migrations are forward-only and
  applied transactionally; a store's catalog, sales, customers and history carry
  forward.
- **Old app versions keep working.** The backend supports the currently published
  app until the next major release forces an update. A new server version will not
  remove an endpoint an older app still calls, except where a major release
  announced it.
- **A minor release never needs a store owner to do anything** for existing
  features to keep working.

## Release notes

Every published release gets notes in [`CHANGELOG.md`](CHANGELOG.md), written for
a shop owner rather than for a developer: what is new, what changed, and what to
do about it. If a release requires action, the action is in the notes and is
repeated on the Play listing.

The Google Play release-notes field is capped at 500 characters, so the Play text
is a summary and this changelog is the full record.

## How a release happens

1. Version string and `versionCode` bumped; changelog entry written.
2. Release artifact signed with the upload key and aligned for 16 KB page
   sizes; permissions and the baked URLs checked across all ABIs.
3. Deployed to production behind a mandatory post-deploy smoke gate that fails
   the deploy rather than warning.
4. Uploaded to Play. The listing, screenshots and release notes are updated in the
   same pass.
5. Staged rollout, watching crash and error telemetry before widening.
