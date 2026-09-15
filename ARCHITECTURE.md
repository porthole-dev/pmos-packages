# Architecture

How packages get from a pmaports branch into this repository, and why it is
built this way.

## Sources and channels

The source of every package is a branch of
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports), a fork of
postmarketOS pmaports. Today there is one: `taimen-bringup`, on the edge
channel, used with systemd.

A fork keeps the upstream `pkgver` and uses `pkgrel` 50 or higher. apk installs
the highest version it can see, so the fork wins over the stock package with
the same `pkgver`, and loses silently once upstream ships a newer `pkgver`.
The weekly upstream check exists to catch that moment.

## One release per apk repository

pmbootstrap builds into two local repositories, and each is published as one
GitHub release whose tag is the path apk requests:

| pmbootstrap local repository | aports | release tag | pmbootstrap option |
| --- | --- | --- | --- |
| `edge` | everything except `extra-repos/systemd` | `main/aarch64` | `mirrors.pmaports_custom` = `.../releases/download` |
| `systemd-edge` | `extra-repos/systemd` (phosh, ...) | `systemd/main/aarch64` | `mirrors.systemd_custom` = `.../releases/download/systemd` |

pmbootstrap turns a mirror into `<mirror>/<branch_pmaports>` and apk appends
`/<arch>/APKINDEX.tar.gz` (`pmb/helpers/repo.py`). `branch_pmaports` for edge
is `main` in the current channels.cfg; it was `master` before, which is why an
early hand-published release is tagged `master/aarch64`. A release channel
such as v26.06 would get `v26.06/aarch64`.

A tag containing `/` works in release download URLs: GitHub serves
`releases/download/<a>/<b>/<file>` for the tag `<a>/<b>` (checked against a
public repository with such tags).

## Prebuilt first

pmbootstrap skips building an aport whose exact version is already in a binary
repository. With this repository configured as a mirror and
`build_pkgs_on_install` off, a user gets every published package prebuilt and
pmbootstrap builds only what is missing: firmware, or a fork changed since the
last publish. Nothing special is needed on the user's side.

CI uses the same mechanism: once this repository is readable anonymously, each
build job configures it as a mirror, so a fork that another fork needs at build
time is downloaded instead of rebuilt.

## Tiers

`.github/packages.conf` on the pmaports branch lists the forks the branch
carries long-term:

- `required`: built whenever its exact version is missing from the published
  repository, even if the push did not touch it.
- `heavy`: multi-hour builds (WebKit, Epiphany, Chromium). Built only when the
  Build workflow is started by hand with *heavy* set, or on a self-hosted runner.

Aports that are not listed are still built when a push changes them.

## Firmware is never published

`firmware-*` packages contain vendor blobs that are not ours to redistribute.
They are excluded in code, not configuration, at three points: the build job
never builds them and deletes any that appear, the build uses pmbootstrap's
`--ignore-depends` so the device package does not pull firmware in, and the
publish step refuses any package named `firmware-*` or containing files under
`lib/firmware`.

## Workflows

All in `.github/workflows/` on the pmaports branch.

| workflow | trigger | jobs |
| --- | --- | --- |
| CI | push, pull request | commit trailer check (DCO sign-off on pull requests), APKBUILD lint with a version bump check, sources and patches check for changed aports |
| Build | push, pull request, manual | select packages; build each on an aarch64 runner; publish (pushes to `taimen-bringup` only) |
| Upstream check | weekly, manual | sources and patches of every listed fork; one issue listing forks an upstream package outranks |

pmbootstrap runs in a privileged Alpine container, prepared the way pmaports'
own CI prepares one (`.github/scripts/run-pmbootstrap.sh`), and pinned to a
commit of the porthole-dev pmbootstrap fork. Builds are native aarch64 on
GitHub's `ubuntu-24.04-arm` runners, so no emulation or cross-compilation is
involved. The checkout needs no secret; only publishing does.

"Sources and patches" runs abuild's own `fetch`, `verify`, `unpack` and
`prepare` through pmbootstrap, so a checksum that changed or a patch that no
longer applies fails in about a minute, before any compiler runs.

## Publishing

For each local repository with new packages, the publish job:

1. downloads the release's current packages;
2. adds the new ones; a package whose file name is already published is kept
   as it is (published versions are immutable), and older versions of the same
   package are dropped;
3. rebuilds `APKINDEX.tar.gz` with `apk index` and signs it with `abuild-sign`
   and the repository key;
4. verifies the signature against the committed public key before uploading;
5. uploads new packages, then the index, then deletes superseded packages, so
   the published index never names a missing file;
6. downloads the index back, checks it is byte-identical and correctly signed,
   and that every package it names is a release asset.

Every package must carry `packager = Giuseppe Maggio <jertlok@proton.me>`;
CI sets it and publish refuses anything else.

## Trust

- The index is signed with the repository key, whose public half is
  `keys/porthole-dev-packages-20260915.rsa.pub`.
- Packages are signed by a throwaway key generated in each build job. apk
  does not need that key: for a package from a repository it checks the
  package against the checksum in the signed index (verified with apk-tools
  3.0.7: a package signed by an unknown key installs from an index signed by a
  trusted key, and the same index is rejected without that key).
- The private key exists only as the `PACKAGES_SIGNING_KEY` secret of the
  `publish` environment in the pmaports repository, which only
  `taimen-bringup` can deploy to. Only the publish job uses it, and that job
  runs no package build code.
- `PACKAGES_TOKEN` (same environment) is a fine-grained token that can write
  this repository's contents (releases) and nothing else. While this
  repository is private, an optional read-only `PACKAGES_READ_TOKEN`
  repository secret lets the select job see what is already published.

## Private now, public later

| | private (now) | public |
| --- | --- | --- |
| install from the releases | `gh release download`, then a local repository | directly, as in the README |
| CI build dependencies | forks rebuilt in each job | downloaded from this repository |
| select "already published" | needs `PACKAGES_READ_TOKEN` | anonymous |
| Actions minutes | 2,000 a month, 2-vCPU arm runners | unlimited, 4-vCPU arm runners |
